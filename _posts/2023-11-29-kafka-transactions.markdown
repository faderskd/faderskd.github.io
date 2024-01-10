---
layout: post
title:  "[Deep dive] Kafka transactions"
date:   2023-11-29
---

# Why Kafka needs transactions ?
The most popular Kafka delivery mode is at least once. It means that it is tuned for reliability - each message will be persisted
in the log without data loss. The downside is possibility of duplicates. In case of publishing failure the producer will retry. 
The same applies to broker failures. While it provides background for many types of application, for some it is not
enough. What if my app has stronger consistency requirements ? What if I need to ensure that the group of messages will be 
persisted exactly once, so all of them are appended to the log or none of them ?
That's why transactions were introduced to the Kafka. The main beneficiary was Kafka Streams. The motivation for it is
broadly described in the [Kafka Transactions proposal](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging#KIP98ExactlyOnceDeliveryandTransactionalMessaging-Status),
but for the sake of this post lets come up with an example similar to transactions in SQL databases.

# Problem statement
Let's say we have an online shop app, with three Kafka topics: `purchase`, `invoices`, `shipments`. Whenever the customer buys
a product, the `purchase` event is produced. It is then processed and two other events are produced:
`shipment` and `invoice`. 

![three-topics.png](/img/transactions/three-topics.png)

Obviously we don't want the situation where we make a shipment without processing invoice. We don't want the product
to be sent twice too. The `purchase` and `invoice` events should be published together (as a unit) or none of them.

# Idempotent producer - Eliminating duplicates
Let's focus on eliminating duplicates first. [I've created an example](TODO://link-do-repo) processing realizing above scenario.
`KafkaConsumer` polls messages from Kafka. The message is then processed and two other events are produced. At the end the consumer
commits the offset to the Kafka, so we don't process the same message twice:

```java
try (KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);
             KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singleton(PURCHASE_TOPIC));
            while (true) {
                try {
                    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                    for (ConsumerRecord<String, String> record : records) {
                        Purchase purchase = mapper.readValue(record.value(), Purchase.class);
                        Invoice invoice = new Invoice(purchase.userId(), purchase.productId(),
                                purchase.quantity(), purchase.totalPrice());
                        Shipment shipment = new Shipment(UUID.randomUUID().toString(), purchase.productId(),
                                purchase.userFullName(), "SomeStreet 12/18, 00-006 Warsaw", purchase.quantity());
                        producer.send(new ProducerRecord<>(INVOICES_TOPIC, mapper.writeValueAsString(invoice))).get();
                        producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, mapper.writeValueAsString(shipment))).get();
                        consumer.commitSync();
                        logger.info("Successfully processed purchase event: {}", purchase);
                    }
                } catch (Exception ex) {
                    logger.error("Error while processing purchase", ex);
                }
            }
        }
```

// TODO: add info when it is enabled
Kafka has had an idempotent producer for a long time. Every time the idempotent producer
sends a message (actually a batch of messages) it remember a sequence number of last message sent.

// TODO: check how to check producer proper idempotence configuration (maybe broker metrics)
Every time a new one is produced the sequence is increased. The new message means separated call to `producer.send(...)`.
In case of any sending retries the message sent twice with the same sequence will be rejected by a broker. Does it mean only one message with
particular sequence can be stored in a log ? 
To understand what's going on, we'll investigate the state kept by both the producer and the broker when enabled idempotent producer.

![idempotent-producer.png](/img/transactions/idempotent-producer.png)

The producer keeps ProducerId (PID) and mapping between TopicPartition and last produced sequence number. The PID is fetched
from the broker (1). Producer sends messages together with its PID and sequence number (2). The first message batch sent creates entry 
in broker's mapping. Every next message successfully sent increments sequence number both on the producer and the broker side. 
This way the broker knows what next sequence to expect. Let's go through possible scenarios and the behaviour of both components:  

1. `ProduceRequest timeout` - while sending batch, Kafka producer may fail to receive a response withing a configured time `request.timeout.ms`. 
What if the broker saved the message, but some transient network issue caused not receiving the response ? The producer will retry 
the message but will use the same sequence number. As we said previously the broker remembers which sequence numbers was sent, so it
knows what to expect next. If the first try saved a message, the broker expects higher sequence number, and rejects lower sequences.
From the producer perspective the request failed, so it uses the same sequence, but from the broker it was successful, and it increases the
next expected sequence for this particular (PID, topic, partition). The idempotence mechanism work properly here.
2. `Producer restart` - our application publishing messages may restart or crash at any time. What if the producer sent the message,
it was persisted by the broker and just before receiving the response the application was restarted ? The new producer will be 
created, it will send `InitPidRequest` and gets new PID. In our example app we commit offset after successful publish, 
we didn't do that before the restart. We reprocess last message, send it with a completely new PID and sequence and we have duplicate. 
The idempotence does not work here. 
3. `Broker restart` - another scenario is when we publish message to Kafka, it is persisted in broker, but it restared/failed just 
before the response was sent back to the producer. To enable idempotence mechanism we have to  

# Transactions usage from client perspective

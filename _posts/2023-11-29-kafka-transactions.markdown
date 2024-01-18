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
// TODO: info about message treated like a batch.
Let's focus on eliminating duplicates first. [I've created an example](TODO://link-do-repo) processing realizing above scenario.
`KafkaConsumer` polls messages from Kafka. The message is then processed and two other events are produced. At the end the consumer
commits the offset to the Kafka, so we don't process the same message twice:

```java
public void start() {
    try {
        consumer.subscribe(Collections.singleton(PURCHASE_TOPIC));
        while (running) {
            try {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
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
    } finally {
        logger.info("Closing consumer...");
        consumer.close();
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
from the broker (1). Producer sends messages together with its PID and base sequence number (2). The base sequence number 
is a sequence of the first message in the batch. The first message batch sent creates entry in broker's mapping. Every next 
message successfully sent increments sequence numbers both on the producer and one the broker side. This way the broker knows 
what next sequence to expect (lastSequence + 1). Let's go through possible scenarios and the behaviour of both components:  

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
3. `Broker restart` - another scenario is when we publish message to Kafka, it is persisted in broker, but the broker restarted/failed just 
before the response was sent back to the producer. Let's see what producer configuration is needed to enable idempotence.
```java
props.setProperty(ProducerConfig.ACKS_CONFIG, "all"); // this must be set to `all`, if not specified, it will be default for idempotence 
props.setProperty(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // this enables idempotence
```
Idempotent producer enforces `ACK all`. It means that every message published to the topic will be fully replicated to the
configured number of in-sync replicas before returning successful response to the client. This is not a post about replication 
so for those who are unfamiliar with ACK-s lets state that the message has to be replicated for some number of replicas.
We previously said, that every message batch sent contains PID and sequence number. These are persisted in the log and replicated.
With every new message replicated from leader, replicas build the same in-memory state. When the current leader fails,
the new one elected will have the state fully replicated (because of `ACK all`).

![idempotence-broker-failure](/img/transactions/idempotence-broker-failure.png)

// TODO: add info about markers
```shell
Dumping /tmp/kraft-combined-logs/shipments-0/00000000000000000000.log
Log starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 6004 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1705529669807 size: 227 magic: 2 compresscodec: none crc: 3294951759 isvalid: true

```

So in case of retry and broker failure the idempotence prevents from duplicates. If the message was persisted on leader, it
was replicated to the followers (2). When the producer retries to a new leader, with previously saved sequence number, it already
has proper in-memory state build from the log. It will reject duplicate. On the producer side the `DuplicateSequenceException` is thrown,
so the producer knows it can skip the message. 

![idempotence-fencing](/img/transactions/idempotence-fencing.png)

# Why idempotent producer if not enough ?
As you see the idempotent producer is not a reliable solution for our case. It only prevents from duplicates in a single
producer session. The producer restart resets its state. Additionally, we still have no way of atomic writes to output topics. 
We need stronger guarantees. We need Kafka transaction mechanism. Once again, before going into details and limitations 
we'll try to play with it. 

# Transactions usage from client perspective

The verbose version: 

```java
@Override
public void start() {
    try {
        producer.initTransactions();
        consumer.subscribe(Collections.singleton(PURCHASE_TOPIC));
        while (running) {
            try {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
                for (ConsumerRecord<String, String> record : records) {
                    Purchase purchase = mapper.readValue(record.value(), Purchase.class);
                    Invoice invoice = new Invoice(purchase.userId(), purchase.productId(),
                            purchase.quantity(), purchase.totalPrice());
                    Shipment shipment = new Shipment(UUID.randomUUID().toString(), purchase.productId(),
                            purchase.userFullName(), "SomeStreet 12/18, 00-006 Warsaw", purchase.quantity());

                    producer.beginTransaction();
                    producer.send(new ProducerRecord<>(INVOICES_TOPIC, mapper.writeValueAsString(invoice))).get();
                    producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, mapper.writeValueAsString(shipment))).get();
                    Map<TopicPartition, OffsetAndMetadata> offsets = Map.of(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset()));
                    producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
                    producer.commitTransaction();
                    logger.info("Successfully processed purchase event: {}", purchase);
                }
            } catch (Exception ex) {
                logger.error("Error while processing purchase", ex);
            }
        }
    } finally {
        logger.info("Closing consumer...");
        consumer.close();
    }
}
```
The refined version: 

```java
producer.initTransactions();
...
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
for (ConsumerRecord<String, String> record : records) {
    ...
    producer.beginTransaction();
    producer.send(new ProducerRecord<>(INVOICES_TOPIC, mapper.writeValueAsString(invoice))).get();
    producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, mapper.writeValueAsString(shipment))).get();
    ...
    producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
    producer.commitTransaction(); // we can call producer.abortTransaction() too 
```

The code does similar thing as in our first approach to the problem with idempotent producer. This time though it does 
it in a consistent manner. It works like this:
1. Initiate the producer and broker to enable transactions. 
2. For each `purchase` event fetch by a consumer, produce `invoice` and `shipment` events to the output topics. 
3. Add `purchases` consumer offset to the transaction.
4. Commit - now all 3 things (2 events and 1 offset move) will be atomically saved to the Kafka. If something wrong 
goes here the transaction will be aborted (we will see it later).










```shell
root@kafka2:/tmp/kraft-combined-logs/shipments-1# /opt/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.log
Dumping 00000000000000000000.log
Log starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 7006 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: true isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1705565397008 size: 227 magic: 2 compresscodec: none crc: 3397250862 isvalid: true
baseOffset: 1 lastOffset: 1 count: 1 baseSequence: -1 lastSequence: -1 producerId: 7006 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: true isControl: true deleteHorizonMs: OptionalLong.empty position: 227 CreateTime: 1705565397153 size: 78 magic: 2 compresscodec: none crc: 1782513362 isvalid: true

```
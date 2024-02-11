---
layout: post
title:  "[Deep dive] Kafka transactions"
date:   2023-11-29
---

### Why Kafka needs transactions ?
The most popular Kafka delivery mode is at least once. It means that it is tuned for reliability - each message will be persisted
in the log without data loss. The downside is possibility of duplicates. In case of publishing failure the producer will retry. 
The same applies to broker failures. While it provides background for many types of application, for some it is not
enough. What if my app has stronger consistency requirements ? What if I need to ensure that the group of messages will be 
persisted exactly once, so all of them are appended to the log or none of them ?
That's why transactions were introduced to the Kafka. The main beneficiary was Kafka Streams. The motivation for it is
broadly described in the [Kafka Transactions proposal](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging#KIP98ExactlyOnceDeliveryandTransactionalMessaging-Status),
but for the sake of this post lets come up with an example similar to transactions in SQL databases.

### Problem statement
Let's say we have an online shop app, with three Kafka topics: `purchase`, `invoices`, `shipments`. Whenever the customer buys
a product, the `purchase` event is produced. It is then processed and two other events are produced:
`shipment` and `invoice`. 

![three-topics.png](/img/transactions/three-topics.png)

Obviously we don't want the situation where we make a shipment without processing invoice. We don't want the product
to be sent twice too. The `purchase` and `invoice` events should be published together (as a unit) or none of them.

### Eliminating duplicates

[//]: # (// TODO: info about message treated like a batch.)

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
                    // We are blocking, then commit for simplicity. We should commit asynchronously.
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

### Idempotent producer
Kafka has had an idempotent producer for a long time. It prevents from duplicated published events. Unfortunately, it happens 
only in a single producer session. Nevertheless, it is a good staring point from which we can progress. 

#### Enabling idempotence 

```java
props.setProperty("acks", "all"); // this must be set to `all`, if not specified, it will be default for idempotence 
props.setProperty("enable.idempotence", "true"); // this enforces enabling idempotence
```

The native Java client has the idempotence enabled by default, if there are no conflicting configurations.
To enforce idempotence set `enable.idempotence` to true. It will then validate and enforce proper configuration. 
Idempotent producer enforces `ACK all`. We will cover why when describing guarantees section. 

#### Producer and broker internal state
Every time the idempotent producer sends a message (actually a batch of messages) it remember a sequence number of last message sent.

[//]: # (// TODO: check how to check producer proper idempotence configuration: log  )

Every time a new one is produced the sequence is increased. The new message means separated call to `producer.send(...)`.
In case of any sending retries the message sent twice with the same sequence will be rejected by a broker. Does it mean only one message with
particular sequence can be stored in a log ? 
To understand what's going on, we'll investigate the state kept by both the producer and the broker when enabled idempotent producer.

![idempotent-producer.png](/img/transactions/idempotent-producer.png)

The producer keeps ProducerId (PID) and mapping between TopicPartition and last produced sequence number. The PID is fetched
from the broker (1). Producer sends messages together with its PID and base sequence number (2). The base sequence number 
is a sequence of the first message in the batch. The first message batch sent creates entry in broker's mapping. Every next 
message successfully sent increments sequence numbers both on the producer and one the broker side. This way the broker knows 
what next sequence to expect (lastSequence + 1). Let's go through possible scenarios and the behaviour of both components.  

#### Guarantees in failures scenarios

1. `ProduceRequest timeout` - while sending batch, Kafka producer may fail to receive a response withing a configured time `request.timeout.ms`. 
What if the broker saved the message, but some transient network issue caused not receiving the response ? The producer will retry 
the message but will use the same sequence number. As we said previously the broker remembers which sequence numbers was sent, so it
knows what to expect next. If the first try saved a message, the broker expects higher sequence number, and rejects lower sequences.
From the producer perspective the request failed, so it uses the same sequence, but from the broker it was successful, and it increases the
next expected sequence for this particular (PID, topic, partition). The idempotence mechanism is sufficient here.
2. `Producer restart` - our application publishing messages may restart or crash at any time. What if the producer sent the message,
it was persisted by the broker and just before receiving the response the application was restarted ? The newly created producer 
sends `InitPidRequest` and gets new PID. In our example app we commit offset after successful publish, we didn't do that before the restart. 
We reprocess last message, send it with a completely new PID and sequence and we have duplicate. This time the idempotence is not enough.  
3. `Broker restart` - another scenario is when we publish message to Kafka, it is persisted in broker, but the broker restarted/failed just 
before the response was sent back to the producer. Let's see what producer configuration is needed to enable idempotence.

```java
props.setProperty("acks", "all"); // this must be set to `all`, if not specified, it will be default for idempotence 
props.setProperty("enable.idempotence", "true"); // this enforces enabling idempotence
```
We said before the idempotent producer enforces `ACK all`. It means that every
message published to the topic will be fully replicated to the configured number of in-sync replicas before returning 
successful response to the client. This is not a post about replication so for those who are unfamiliar with ACK-s lets 
state that the message has to be replicated for some number of replicas. We previously said, that every message sent 
contains PID and sequence number. These are persisted in the log and replicated. With every new message replicated from leader, 
replicas build the same in-memory state. When the current leader fails, the new one elected will have the state fully 
replicated (because of `ACK all`).

![idempotence-broker-failure](/img/transactions/idempotence-broker-failure.png)

// TODO: add info about markers
```shell
Dumping /tmp/kraft-combined-logs/invoices-0/00000000000000000000.log
Log starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 6004 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1705529669807 size: 227 magic: 2 compresscodec: none crc: 3294951759 isvalid: true

```

So in case of retry and broker failure the idempotence prevents from duplicates. If the message was persisted on leader, it
was replicated to the followers (2). When the producer retries to a new leader, with previously saved sequence number, it already
has proper in-memory state build from the log. It will reject duplicate. On the producer side the `DuplicateSequenceException` is thrown,
so the producer knows it can skip the message. 

![idempotence-fencing](/img/transactions/idempotence-fencing.png)

#### Why idempotent producer if not enough ?
As you see the idempotent producer is not a reliable solution for our case. It only prevents from duplicates in a single
producer session. The producer restart resets its state. Additionally, we still have no way of atomic writes to output topics. 
We need stronger guarantees. We need Kafka transaction mechanism. Once again, before going into details and limitations 
we'll try to play with it. 

### Transactions

Before we get into guarantees Kafka transactions give, I find it helpful to understand upfront how they work. For now 
let's state that they provide atomic consume, transform, publish loop. It means that we can get event(s) from a Kafka topic, 
map it to another event(s) and publish mapped value(s) to another topic(s) as a single unit. If any of the three steps fails 
(or we abort explicitly) nothing happens from the Kafka client's perspective. 

#### Usage from client perspective

The verbose version: 

```java
public void start() {
    try {
        producer.initTransactions();
        consumer.subscribe(Collections.singleton(PURCHASE_TOPIC));
        while (running) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
            for (ConsumerRecord<String, String> record : records) {
                try {
                    Purchase purchase = mapper.readValue(record.value(), Purchase.class);
                    String invoiceEvent = mapper.writeValueAsString(new Invoice(purchase.userId(), purchase.productId(),
                            purchase.quantity(), purchase.totalPrice()));
                    String shipmentEvent = mapper.writeValueAsString(new Shipment(UUID.randomUUID().toString(), purchase.productId(),
                            purchase.userFullName(), "SomeStreet 12/18, 00-006 Warsaw", purchase.quantity()));

                    producer.beginTransaction();
                    producer.send(new ProducerRecord<>(INVOICES_TOPIC, invoiceEvent));
                    producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, shipmentEvent));
                    Map<TopicPartition, OffsetAndMetadata> offsets = Map.of(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1));
                    producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
                    producer.commitTransaction();
                    logger.info("Successfully processed purchase event: {}", purchase);
                } catch (JacksonException ex) {
                    logger.error("Error parsing purchase message", ex);
                } catch (Exception ex) {
                    logger.error("Error while processing purchase", ex);
                    producer.abortTransaction();
                    resetToLastCommittedPosition(consumer);
                    break;
                }
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
    producer.send(new ProducerRecord<>(INVOICES_TOPIC, mapper.writeValueAsString(invoice)));
    producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, mapper.writeValueAsString(shipment)));
    ...
    producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
    producer.commitTransaction(); // or we can call producer.abortTransaction() too 
```

The code does similar thing as in our first approach to the problem with idempotent producer. This time though it does 
it in a consistent manner. It works like this:
1. Initiate the producer and broker to enable transactions. 
2. For each `purchase` event fetch by a consumer, produce `invoice` and `shipment` events to the output topics. 
3. Add `purchases` consumer offset to the transaction.
4. Commit - now all 3 things (2 events and 1 offset move) are atomically saved to the Kafka, no partial results. If something wrong 
goes here the transaction is aborted, and the consumers don't see aborted events.

// TODO: add info that we can batch multiple tranactions into one but for simplicity we sequentially process records

### Transaction compontents
Ok, we can go deeper in each part of the transaction. It's time for drawings. 

![transactions-components](/img/transactions/transactions-components.png)

#### Transactional producer
To enable transactional producer we need to turn on idempotency and add additional parameter: `transactional.id`.

```java
props.setProperty("acks", "all");
props.setProperty("enable.idempotence", "true");
props.setProperty("transactional.id", "myProducer");
```

The `transactional.id` is a unique identifier provided by us for a producer, and it has to be the same between producer restarts. 
Otherwise, we don't get transactional guarantees. Ok, but what it means to be the same ? 
In an environment where we have multiple producer instances,
every transactional id must map to a consistent set of input topic partitions. In our case:
1. If topic `purchases` has two partitions [1, 2].
2. And we have two instances of the service consuming from `purchases`, and producing new events to the output `invoices`/`shipments` topics. 
3. One instance is consuming from partition [1] and producing to the both output topics.
4. Second instance is consuming from partition [2] and producing to both output topics.
5. Then instance consuming from partition [1] must have the same transactional id for its producer between restarts. 
6. The same holds for second instance.

#### Transaction coordinator
The `transaction coordinator` - is just a usual broker. Every broker can be coordinator. The coordinator manages transactions for specific producer. 
Just like the SQL databases keeps the WAL (write ahead log) of the transaction's operations, Kafka has transaction log too. It is an internal Kafka 
topic (`__transaction_state`). This topic is divided into 50 partitions. This number suggests that if you have less than or equal to 
50 brokers in a cluster, every broker has its own set of partitions. If you have more (wow!), increase the number of transaction log partitions. 
Transaction log is replicated to the follower replicas for availability in case of coordinator failure. 

#### Initialization

```java
producer.initTransactions();
```

![transactions-initialization](/img/transactions/transactions-initialization.png)

The first thing to do by a producer is to find its transaction coordinator. That's why it sends `FindCoordinatorRequest`(1) to any broker. 
The broker then finds transaction coordinator and returns it to the producer (2). The algorithm is quite simple. It takes the hash from 
producer `transactional.id` and does the modulo using `__transaction_state` topic partitions count. The coordinator will be the leader 
of the calculated partition number. Pseudocode:

```java
var kafkaTransactionLogPartitionId = hash(transactional.id) % transactionStateTopiPartitionsCount;
var coordinatorId = leaderOf(kafkaTransactionLogPartitionId);
```

When the producer knows its coordinator it can send `InitPidRequest` request for acquiring the PID (3). The response (4) contains 
the PID and epoch number. Just like in case of idempotent producer the former prevents from duplicates across single session, 
but the latter is used for distinguishing between multiple incorporates of the same producer. So let's assume we have the following scenario:
1. Producer with `transactional.id` set to `A` sends `InitPidRequest` to the transaction coordinator and gets `PID=2, epoch=0`
2. Some time passed and the producer hung for some reason, so it was considered dead. 
3. The new instance of the producer with the same `transactional.id` (`A`) was spawned. 
4. The new instance sends `InitPidRequest` but this time it gets `PID=2, epoch=1` from the coordinator.
5. When the first instance wakes up and tries to start new transaction it is immediately fenced, because the sent epoch number is 
`0` and the newest is `1` for that transactional id. 

#### Producer and broker internal state

![transactional-state.png](/img/transactions/transactional-state.png)

1. Similarly to the idempotent producer, transactional `KafkaProducer` keeps the mapping from topics partitions to the sequence number. 
Because it has to avoid duplicates even in case of restarts it needs to remember its `transactional.id` and epoch number
increasing with each producer restart. The epoch is used to fencing zombie instances of the old producer with same transactional id.
2. Transaction coordinator keeps mapping from `transactional.id` to the producer PID, epoch number, output topic partitions 
involved in transaction, status and few other fields not included in the picture above. 

#### Staring transaction

```java
producer.beginTransaction();
```

The producer makes a bunch of validation and changes its internal state indicating that the transaction is in-progress. 
No external requests to brokers here. 

#### Sending messages to brokers

```java
producer.send(new ProducerRecord<>(INVOICES_TOPIC, mapper.writeValueAsString(invoice)));
producer.send(new ProducerRecord<>(SHIPMENTS_TOPIC, mapper.writeValueAsString(shipment)));
```

![transactions-sending-messages.png](/img/transactions/transactions-sending-messages.png)

Finally, the messages go to the broker, huh? Well, not quite. Before the producer sends messages to the brokers, it has to 
record which topic partitions it's sending messages to. So it sends `AddPartitionsToTxnRequest`(1) to the coordinator. This information 
will be needed when committing/aborting transaction. The `AddPartitionsToTxnRequest` is sent only once for each topic partition.
The coordinator just has to know which brokers to inform about transaction ending process.
The partitions are added to the coordinator's in-memory map and applied to the transaction log (and replicated).  
Now, the producer can go further and send messages to the brokers (2). The brokers can reject duplicates because of `PID` included (idempotency). 
They can also reject messages from zombie instances because the presence of `epoch`. If the epoch is lower than expected, 
the message is rejected. The brokers recognize if a message is transactional. The messages have special `isTransactional` attribute.  
It will be important when returning the messages to the consumers with proper isolation level. We'll cover that later. 
**Caution**
The actual `ProduceRequest` is way more complex than on the picture. I've included only part of the request for one partition, 
containing data important from the transactions viewpoint. 

#### Sending offsets to consumer coordinator

```java
producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
```

The offset committing in a Kafka is a topic (non pun intended) for a completely new blog post. I assume the reader knows basics
of consumer-groups and offset committing. If not, please read the following [article](https://docs.confluent.io/platform/current/clients/consumer.html).

We've consumed message from the `purchases` topic and published `invoice/shipment` events to the target topics. 
These are still not visible to the consumers. Now we want to move forward on the `purchases` topic and commit consumed 
offset. This fact has to be recorded to the transaction log and being part of the transaction. Why ? Once again, to get 
atomicity and avoids duplicates. What if we consumed the `purchase` event, produced atomically outgoing events to the target topics 
and before committing the offset for `purchases` the app crashed ? After recovery, the app has to start from some point. 
It is the last committed offset before crash. But this one is some before the last message consumed. The Kafka solves that problem
by confining offset commit as a part of transaction: message consuming, producing and offset committing happens together or 
none of them.

![transactions-committing-offsets.png](/img/transactions/transactions-committing-offsets.png)

When adding offsets to transaction the producer sends `AddOffsetsToTxnRequest` (1). It contains basic transactional fields which 
we've previously mentioned, and one additional field - consumer `groupId`. Base on this, the transaction coordinator calculates
`__consumer_offsets` topic partition for our of consumer group (the consumer group of the `purchases` topic consumer). 
You already know the algorithm. It is similar to one used in finding transactional coordinator.
```
var kafkaConsumerOffsetsTopicPartitionId = hash(purchasesConsumerGroupId) % consumerOffetsTopicPartitionsCount;
```
As usual, it is added to the transaction coordinator state, and replicated via internal `__transaction_state` topic. 
The calculated partitions will be then used when committing/aborting transaction. 

We notified the transactional coordinator about offsets, now it's time to notify the consumer group coordinator. 
Under the hood`KafkaProducer` sends another `TxnOffsetCommitRequest` (2) to our consumer's coordinator. How to find a consumer group 
coordinator ? 

```
var kafkaConsumerOffsetsTopicPartitionId = hash(purchasesConsumerGroupId) % consumerOffetsTopicPartitionsCount;
var coordinatorId = leaderOf(kafkaConsumerOffsetsTopicPartitionId);
```

The request contains information about consumed topics partitions' offsets. In our case obviously about `purchases` topic. 
The offset if appended to the `__consumer_offsets` topic, but it is not returned to the clients until the transaction commits. 
// TODO check markers for __consumer_offsets topic

Phew, quite a lot of work in a single function call. This is not the end. We still have a lot to cover.

#### Commit or abort

```java
producer.commitTransaction(); // or we can call producer.abortTransaction() too 
```

The last step is ending transaction. As in databases world we can commit or abort. The commit materializes the transaction,  
and consumers of the output topics can see its result. The abort makes all events in being part of the transaction invisible. 
However, there is one requirement: you need to properly configure consumer. Before that, let's see what happens in the brokers. 

![transactions-markers.png](/img/transactions/transactions-markers.png)

The producer sends a `EndTxnRequest` (1) to the transaction coordinator. As usual, the request contains `transactionalId`, 
`PID`, `epoch` and a marker `committed` indicating if to commit or abort. As usual the request contains some other fields, I've not covered 
in this post. The coordinator starts two phase commit.
In the first phase, it updates its internal state and writes the `PREPARE_COMMIT/PREPARE_ABORT` 
messages to the transaction log. The massage is fully replicated and then the response is returned to the producer. At this point 
we can be sure the transaction is committed/aborted no matter what. If the coordinator crashes it can be replaced by a replica 
having the transaction log copy mirroring the current transaction coordinator state. 
The second phase is writing `COMMIT/ABORT` transaction markers to the leaders of topics partitions included in the transaction itself (2, 2', 2''). 
In our case they are `shipments`, `invoices` and `__consumer_offsets` topics. The markers are appended via the `WriteTxnMarkerRequest` request. 
After that, the messages being part of the transaction can be returned to the consumers now. 
The consumers will handle the messages and markers properly based on the configured isolation level.
Note that this is the first time when the coordinator communicates with the leader brokers. Because the offsets topic is also 
a part of the transaction it has to write marker to the consumer group coordinator of our consumer fetching from `purchases` topic. 
We know how to find it because we recorded `__consumer_offets` partition in the transaction log when sending offsets to transaction. 
Depending on the marker type, the consumer group coordinator will return proper offset. In case of the abort it returns offsets 
before the transaction started, in case of commit it has to return offsets included in transaction itself. This way we don't 
reprocess the same `purchases` events in case of committed transaction.

### Transactional consumer

We've already covered the producing side of exactly once delivery. As mentioned before, the consumer has its role in the 
process too.

#### Usage from client perspective

```java
consumerProps.setProperty("group.id", "purchasesConsumerGroupId");
consumerProps.setProperty("enable.auto.commit", "false");
consumerProps.setProperty("isolation.level", "read_committed");
```
As usual, we have to provide consumer group id. We disable the offset auto-committing because it is a part of the transaction 
managed by transactional producer. The isolation level controls how to read messages written transactionally. From documentation: 
"If set to `read_committed`, consumer.poll() will only return transactional messages which have been committed. If set to `read_uncommitted`, 
consumer.poll() will return all messages, even transactional messages which have been aborted." The transaction markers are not 
returned to the applications in either case. 
The important thing is that even in `read_committed` mode there is no guarantee that a single consumer reads 
all messages from single transaction. It consumes transactional committed messages only from the assigned partitions. Transaction 
can span multiple partitions from multiple topics, part of them can be handled by different consumer instance (image below).

![transactions-two-consumers.png](/img/transactions/transactions-two-consumers.png)

#### Broker's LSO and consumer index

The messages returned to the apps are always in their offset order. In `read_committed` mode, the consumer has to know upfront which 
messages are aborted, so they are not exposed to the client. Take the following log as an example:

![transactions-log-of-messages.png](/img/transactions/transactions-log-of-messages.png)

While consuming messages in order, the consumer can't return messages with offsets 0 and 1. In case of abort, 
it has to omit the offsets being part of aborted transaction. In other words, the consumer needs some aborted messages filter. 
This is the purpose of aborted transaction index. Thanks to this structure, the consumers know where are the offsets of aborted messages, 
so `read_committed` mode can filter them out. The index is build on the broker side, and returned to the consumer in the `FetchResponse` 
together with ordinary messages data.

![transactions-aborted-index.png](/img/transactions/transactions-aborted-index.png)

The on-disk index structure:

```
root@kafka2:/tmp/kraft-combined-logs/invoices-0# /opt/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.txnindex 
Dumping 00000000000000000000.txnindex
version: 0 producerId: 0 firstOffset: 0 lastOffset: 1 lastStableOffset: 2
```

While returning messages, the broker returns only resolved transactions to the `read_committed` consumer. This means that any 
pending transactions (without COMMIT/ABORT markers) will not be visible by the consumers until they end. To maintain such an invariant, 
the broker keeps LSO (last stable offset) in its memory. It indicates the offset below which all transactions have been resolved. 
Without this, it would not be possible to build the transaction index, as pending transactions has no markers written yet.

![transactions-lso.png](/img/transactions/transactions-lso.png)

### Full transactional flow

As we already know all the transaction stages it is convenient to have a full picture in one place.

![transactions-full-picture.png](/img/transactions/transactions-full-picture.png)

That's a lot of steps, let's cover them in detail. 

1. `KafkaProducer.initTransactions()` is called. The producer is being part of `ClientProducingApp` application. 
The `FindCoordinatorRequest()` is sent to any broker (Broker5) on the picture. The request contains `transactionalId` used 
in calculating the transaction coordinator (Broker4).
2. `KafkaProduer` sends `InitPidRequest` to the coordinator, in response it gets producer id `PID` with some additional fields 
just like producer epoch. This data is used both for idempotency and transactions API. It is also stored in brokers' log. 
3. `KafkaConsumer.poll()` is called. The consumer is being part of the same `ClientProducingApp` application. The app 
performs some atomic consume-transform-produce loop. It consumes events from `purchases` topic and produces other events to 
`invoices` and `shipments` topics as a single atomic unit. Obviously the app can be scaled itself, but for the sake of better 
visibility we have only one instance. The app makes separated transaction for each record it gets from `purchases` topic. Again, multiple 
records can be batched in a single transaction for high throughput, but I'd like to focus on simple, easier to grasp scenario.  
4. `KafkaProducer.beginTransaction()` is called. No external requests, only internal producer state is changes. 
5. `KafkaProducer.send()` method is called. Producer calculates target topic partition. It knows on which broker it is 
situated based on the internal metadata taken previously from the brokers. If it is the first message to this specific topic 
partition, the producer sends `AddPartitionsToTxnRequest()` to the transaction coordinator. The coordinator updates its 
current transactions internal state, persist the information from request to the `__transaction_state` topic and waits for 
replication to follower replicas. 
6. As part of the same `KafkaProducer.send()` method from previous point, producer sends the `ProduceRequest` with messages 
to the leaders of calculated topic partitions. In our case these are Broker2 for `invoices` and Broker3 for `shipments` topic. 
The messages are not sent immediately with a `send` call because producer is doing batching and sends them asynchronously. 
Nevertheless, they will be sent at some point. Besides the records itself, the request contain `PID`, epoch and some other fields. 
It contains the `transactionalId` too, but only for authorization purposes. The `transactionalId` is not persisted in the 
topic data (`invoices`/`shipments`) logs. 
7. `KafkaProducer.sendOffsetsToTransaction()` is called. Producer sends `AddOffsetsToTxnRequest` to transaction coordinator with 
a consumer group id as one of the fields. The coordinator calculates offset topic partition, updates its internal state and
replicates the state via `__transaction_state` topic to the followers. 
8. As part of the same `KafkaProducer.sendOffsetsToTransaction()` method from previous point, producer calculates the 
`purchasesConsumerGroupId` consumer group coordinator. The consumer group is used by `ClientProducingApp`. 
Producer then sends the `TxnOffsetCommitRequest` to the calculated coordinator. The request contains information about offsets 
at `purchases` topic partitions for the consumer group. Consumer group coordinator appends offsets to its internal 
`__consumer_offsets` topic but does not it offsets in-memory state yet. It can't return the new offsets to clients until the 
transaction commits. 
9. `KafkaProducer.commitTransaction()` or `KafkaProducer.abortTransaction()` is called. Producer sends `EndTxtRequest` to 
the transaction coordinator. Coordinator starts two phase commit. Firstly it updates its internal state by changing it to the 
`PREPARE_COMMIT`/`PREPARE_ABORT`. It persists the state to the log and waits for replication to the followers. 
10. At the second phase, the coordinator sends `WriteTxnMarkerRequest`. It causes writing transaction `COMMIT/ABORT` markers 
to the topic partitions included in transactions. These are partitions for `invoices`, `shipments` and `__consumer_offsets` topics. 
In case of coordinator crash, the information about transaction status is recorded to the `__transaction_state` log, 
so the new coordinator can start from the same point and continue ending transaction process. In case of `__consumer_offsets` topic, 
the transaction coordinator takes consumer group id from its transaction state, calculates the consumer group coordinator and sends 
append marker request. 
11. When the previous point succeeds the transaction coordinator updates its internal transaction state to `COMPLETE_COMMIT`/`COMPLE_ABORT` 
end the transaction is ended. 
12. After successful writes of transaction markers (point 10) to target topics, the brokers can now return new messages 
to the consumers. The brokers move their LSOs (last stable offset) so they include the new messages. 
The client app is consuming from target topics with `read_committed` mode. The call to `KafkaConsumer.poll()` sends `FetchRequest`. 
The response contains new records together with aborted transaction index. The consumer uses the index to filter out aborted 
transactions.
In case of `read_uncommitted` mode, the brokers return new messages to the consumers immediately after they are appended to the 
broker's log via `KafkaProducer.send()` from point 5.

### Guarantees in failures scenarios

Finally, as we know how everything works together we can discuss Kafka exactly one delivery guarantees. 

1. `KafkaProducer.send()` failure
2. `KafkaProducer` failure
3. `KafkaConsumer` failure (the one being part of consume-transform-produce loop)
4. `KafkaConsumer` failure (the one consuming from target topics)
4. Transaction coordinator
5. Broker failure
6. Consumer group coordinator failure

[//]: # (```shell)
[//]: # (root@kafka2:/tmp/kraft-combined-logs/shipments-1# /opt/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.log)
[//]: # (Dumping 00000000000000000000.log)
[//]: # (Log starting offset: 0)
[//]: # (baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 7006 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: true isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1705565397008 size: 227 magic: 2 compresscodec: none crc: 3397250862 isvalid: true)
[//]: # (baseOffset: 1 lastOffset: 1 count: 1 baseSequence: -1 lastSequence: -1 producerId: 7006 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: true isControl: true deleteHorizonMs: OptionalLong.empty position: 227 CreateTime: 1705565397153 size: 78 magic: 2 compresscodec: none crc: 1782513362 isvalid: true)
[//]: # ()
[//]: # (```)
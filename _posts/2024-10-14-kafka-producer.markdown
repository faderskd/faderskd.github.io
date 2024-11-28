---
layout: post
title: "Kafka producer deep dive"
date: 2024-10-14
---

> [INFO]  
> The post covers replication KafkaProducer version (>=3.8).
> {: .block-tip }

### Prerequisites

- [Kafka replication deep dive]({% post_url 2024-05-15-kafka-replication %})

### Kafka protocol

Kafka exposes a set of APIs that clients can use to interact with the broker. The protocol is a request response [binary](https://en.wikipedia.org/wiki/Communication_protocol#Binary)
protocol over TCP. Each request has a unique API key, so the brokers knows how to handle it. This API 
key is something like a method name and API path together in the HTTP protocol. And there is a lot of requests types in Kafka. 
The complete list is [here](https://github.com/apache/kafka/blob/3.8.0/clients/src/main/java/org/apache/kafka/common/protocol/ApiKeys.java#L45). 
In this post we'll focus on the first one - `ProduceRequest`. 

### KafkaProducer

Kafka comes with a producer client that is used to publish messages to the brokers. From a high level perspective, it 
just sends produce requests. Under the hood, there is a sophisticated machinery handling the batching, compression, partitioning, 
retries, metadata refreshing, load balancing... and much, much more. And as usual, we'll cover a lot of practical stuff 
but also some interesting internals. 

### KafkaProducer basic config and usage

#### Basic Configuration and usage

We'll start with something really simple. This is the bare minimum configuration and the simplest usage. 
Don't get this tiny example seriously - this code sucks, but we'll improve it later. 

```java
public class SynchronousAuditProducer implements ProgramLoop {
    private final static Logger logger = LoggerFactory.getLogger(SynchronousAuditProducer.class);
    private final static ObjectMapper mapper = new ObjectMapper();
    private final static String AUDIT_TOPIC = "userActivity";

    private final KafkaProducer<String, String> producer;
    private volatile boolean running = true;

    public SynchronousAuditProducer() {
        this.producer = new KafkaProducer<>(producerProperties());
    }

    private static Properties producerProperties() {
        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093,localhost:9094,localhost:9095");
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        return props;
    }

    @Override
    public void start() {
        try {
            while (running) {
                try {
                    AuditLog p = generateExampleAuditLog();
                    ProducerRecord<String, String> record = new ProducerRecord<>(AUDIT_TOPIC, mapper.writeValueAsString(p));
                    int partition = producer.send(record).get().partition();
                    logger.info("Successfully sent audit event to partition {}", partition);
                } catch (Exception ex) {
                    logger.error("Error while sending audit event", ex);
                }
            }
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }

    @Override
    public void wakeup() {
        logger.info("Program loop wakeup");
        running = false;
    }

    private static AuditLog generateExampleAuditLog() {
        ActionType actionType = ActionType.values()[(int) (Math.random() * ActionType.values().length)];
        String username = "user" + (int) (Math.random() * 100);
        String now = Instant.now().toString();
        return new AuditLog(now, username, actionType);
    }
}
```

`KafkaProducer` needs at least 3 configs:
- [bootstrap.servers](https://kafka.apache.org/38/documentation.html#producerconfigs_bootstrap.servers) - List of brokers in the cluster. 
Kafka producer will send a metadata request to one of the brokers to get the information about rest of the cluster. 
- [key.serializer](https://kafka.apache.org/38/documentation.html#producerconfigs_key.serializer) - Message key serializer. 
Producer sends binary data to the broker, so we need a way to serialize message key (if given) to the array of bytes.
- [value.serializer](https://kafka.apache.org/38/documentation.html#producerconfigs_value.serializer) - Message value serializer, 
the same as above but for the message value. 

The `send(ProducerRecord record)` method returns a `Future<RecordMetadata>`. The `RecordMetadata` contains metadata about the 
produced message - topic, partition, offset, timestamp, etc. The `Future.get()` method blocks current thread until the record is sent. 
This is just a tiny example, use it rather for local debugging than anything else. To get to production-like usage we have to 
cover a lot of stuff. 

#### KafkaProducer send() 10000 foot view

The above configuration is a toy starting point, and it needs a lot of improvements. There are tons of different knobs that 
can be adjusted in the producer. We'll cover the most important ones while explaining how the `send()` method works. 
But for now we will cover the sending flow in a very high level. Then I'll provide you with a full configuration example, 
and only after we'll dive into more details of the producer's internals. 

Basically, the `send(ProducerRecord, ...)` method can be divided into few main steps, each changing its behavior based on 
configuration and parameters given. Look at first parameters of the send method - `ProducerRecord`.  

```java
public class ProducerRecord<K, V> {

    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
    ...
```

Each of these fields you can provide while constructing the producer record, and each of them affects the producer's send behavior. 
This is how `send()` works:
1. Based on the `topic`, the producer knows where (to which brokers) it has to send data.   
2. **Serializing** `key` and `value`. Kafka doesn't care about the type of the key and value. All its wants is the array of bytes. 
The key is used in partitioning (more about that in a moment), and the value is the actual message.  
3. **Partitioning**
   1. If the partition is specified upfront, the producer will send the message to the leader of that partition. Usage of this
      option involves calculating the partition before sending. 
   2. If not, the producer will look for a [custom partitioner](https://kafka.apache.org/documentation/#producerconfigs_partitioner.class) 
      provided by a configuration `props.setProperty(PARTITIONER_CLASS_CONFIG, CutomPartitioner.class.getName())`.
   3. If partitioner class is not provided and the key is present the producer will calculate the partition on its own using
      algorithm like: `hash(key) % numPartitions`. 
   4. If the custom partitioner and the key are not present the partition is chosen using one of the available load balancing 
      strategies. And here we have a few choices too. We will cover that in a later section dedicated to a detailed 
      `send()` method explanation.
4. **Batching**
   1. Instead of sending each message separately the producer will batch messaged for each partition. This will increase 
      throughput as we save on the network round trips times (number of request/response cycles). But obviously, we don't 
      to want indefinitely wait for a batch to fill so we have to trade-off between the batch size and the cutoff time. 
   2. The [batch size](https://kafka.apache.org/documentation/#producerconfigs_batch.size) control how many **bytes** we can send 
      to a single partition. Note that this is not a request size sent to the broker, because the broker owns multiple partitions
      (more about internals later). Default value is 16KB. The configuration of this value should be tested in your specific case. 
      Too small size will increase the number of requests, and decreasing throughput. Too big size will not affect maximum 
      wait time as we have another configuration for that. But it affects memory usage distribution inside the producer. 
      More about this in points 4 and 5. 
   3. [linger.ms](https://kafka.apache.org/documentation/#producerconfigs_linger.ms) is the maximum time the producer will wait 
      for the batch to fill. If the batch is not full after that time, it will be sent anyway. This will protect from 
      unpredictable latencies in case the batches are not full. The default value is 0 which means disabled batching. The 
      reality is that `linger.ms=0` may still batch in an application with high sending throughput. More about it later. 
   4. [buffer.memory](https://kafka.apache.org/documentation/#producerconfigs_buffer.memory) is the maximum memory the producer 
      can use to for messages waiting to be sent. The default value is 32MB. The producer has a memory pool that is given 
      to batches to each partition. Each time a new batch is created/sent the memory is subtracted/reclaimed to this pool.
      If you run out of that memory the producer will block the application calling `send()` until some memory is freed.
   5. How to set these parameters correctly?
      - start with defining requirements for throughput and latency
      - then measure them with different configurations, remembering about below trade-offs (more about metrics later)
      - if you don't care about latency so much you can set larger `batch.size` and `linger.ms` to increase throughput
      - if you care about latency, set `linger.ms` to limit the maximum batching time
      - if your app has low throughput, setting too large `batch.size` and too small `linger.ms` may end up 
        in sending mostly empty batches. Each batch consumes constant memory (`batch.size`) from a memory pool 
        which is not reclaimed until the batch is sent. Running out of memory will block the producer until memory is freed
      - monitor the batching so you can react accordingly - this will be covered in the monitoring section.
5. **Sending**
   1. Once we have batches ready to sent, the producer send them to the respective brokers. It will basically take a bunch of 
   ready batches, check their partitions, find the leaders of that partitions, and group them by those leaders. The result 
   is a map of   
   `{ broker -> [batchesToSendForThatBroker] }`:
   ```java
   {
     broker1 -> [batch1, batch2],
     broker2 -> [batch3, batch4],
     ...
   }
   ```
   2. The producer will then make a `ProduceRequest([list of batches])` to each broker.
   3. And now is time for more configuration options.
   4. [acks](https://kafka.apache.org/documentation/#producerconfigs_acks) - controls how many replicas must acknowledge 
   the record before the producer considers the record as sent. I'll stick with the DRY principle this time and really 
   encourage you to read my post about [Kafka replication]({% post_url 2024-05-15-kafka-replication %}) before.
   It deeply explains how replication works in Kafka and how the `acks` affects the producer's behavior in different 
   scenarios.
   6. [delivery.timeout.ms](https://kafka.apache.org/documentation/#producerconfigs_delivery.timeout.ms)
   5. [max.request.size](https://kafka.apache.org/documentation/#producerconfigs_max.request.size)
   5. [retries](https://kafka.apache.org/documentation/#producerconfigs_retries)
   8. [request.timeout.ms](https://kafka.apache.org/documentation/#producerconfigs_request.timeout.ms)
   7. [max.block.ms](https://kafka.apache.org/documentation/#producerconfigs_max.block.ms)
   8. [enable.idempotence](https://kafka.apache.org/documentation/#producerconfigs_enable.idempotence)
   ![kafka-timeline-vs-timeouts.png]()


 # example with batching
 # example metrics
 # exmaple of measuring e2e latency

##### ACKS

```java
props.setProperty(ProducerConfig.ACKS_CONFIG, "1"); // can be 0, 1, all
```

This control how many replicas must acknowledge the record before the producer considers the record as sent. I'll stick 
with the DRY principle this time and really encourage you to read my post about [Kafka replication]({% post_url 2024-05-15-kafka-replication %}) before. 
It deeply explains how replication works in Kafka and how the `acks` affects the producer's behavior in different scenarios. 
The official config documentation is [here](https://kafka.apache.org/documentation/#producerconfigs_acks).



### High level communication view



![kafka-tcp-protocol.png]()

Before any request can be sent, producer must establish a connection with the broker. But there are many brokers in the 
cluster, and connecting to each of them is not necessary until we have records to send to that broker. 


### KafkaProducer components
// TODO - explain role of each component + diagram

### KafkaProducer send

1. Call interceptors on a record - this can change the record before it will be serialized.
2. Fetch metadata for the topic of the record. If no metadata is available for topic, the producer will add
   the topic to the metadata list and schedule metadata update. The call is blocked until metadata is available. 
   Note that matatada is updated only for interested topics - the ones for which the producer has records to send.
   #TODO - how metadata updating works
3. Serialize the record key and value.
4. Calculate the partition for the record: 
   a) If partition is specified in the record, use it.
   b) If custom partitioner is specified, use it.
   c) If key is present, use the hash of the key.
   d) If key is not present, use return UNKNOWN_PARTITION and delay partition choice.
5. Append record to accumulator:
   1. Get current sticky partition. 
   2. If we don't have load stats
      1. get random available partitions from the metadata (point 2). Available partition is 
      a one that has a leader. 
      2. If we don't have available partitions just get random partition.
   3. If we have load stats, get random partition using probability proportional to the lowest load: the lower the load
      the higher the probability of choosing the partition. The load stats are calculated using cumulative frequency table. 
      It works as follows: (#TODO: How adaptive partitioning fits into this?)  
      a) We have a list of partitions with their loads expressed as diff between max allowed queue (what is queue?) of the partition 
         and current queue size + 1. 
      b) We calculate cumulative frequency table for the partitions.
      c) We choose a random number between 0 and sum of all loads.
      d) We find the partition for which the random number is strictly greater than cumulative frequency.
      e) Example: 
         - We have 3 partitions (0,1,2) with loads: 5, 1, 3. Max queue size is: 7. Random number is 4. 
         - Diffs: [7 - 5 + 1, 7 - 1 + 1, 7 - 3 + 1] = [3, 7, 5]
         - Cumulative frequency: [3, 10, 15]
         - Random number is 4, so strictly greater cumulative frequency is 10 - thus it is partition 1.
   4. Once we have a partition, look for a queue of batches to that partition. Take the last batch, and
      if it's not full, append the record. Record the user provided callback and return a future. That callback will be then 
      called when the record is sent. We'll get to that. 
   5. If the batch is full, we have to create a new one, but to that we need to allocate memory. `KafkaProducer` limits total 
      memory usage to `buffer.memory` property. If we exceed that, we'll block until some memory is freed. Once we have memory,
      we create a new batch. 
   6. During the switch KafkaProducer informs the partitioner to pick the new partition. May exists edge cases when the 
      `linger.ms` passed, but the bytes sent < `batch.size`. In that case, we'll send the batch anyway, but don't immediately change 
      partition. Max 2x `batch.size` can be sent in that case.
6. Sending messages
   1. Once message is appended to the batch, we return a future to the client. The future is completed when the record is sent. 
      If the batch is full we notify the sender so it can start sending the batch to the broker.
   2. The KafkaProducer uses a single io thread to run sender in a loop.  
   3. Each iteration of the loop will try to find and send ready broker nodes -> with partitions -> with batches ready to be sent. 
      The broker is ready if there is at least one partition that is not backing off its send (#TODO when can backoff) 
      and those partitions are not excluded from new messages sending, which is the case when 
      ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION == 1). 
   4. Once we have candidate partition for sending we, one of the following must be true:
      1. The partition has full batch to be sent.
      2. The partition batch's linger time passed.
      3. KafkaProducer is out of memory so it tries to immediately send the batches (even uncompleted) to their partitions 
      and free some memory. 
   5. By the way of calculating ready nodes to send data the KafkaProducer calculates here statistics for adaptive partitioning 
      used in point 5.3.
      - checking if partitions have leaders
      - getting partitions queue size
      - if enabled adaptive partitioning with incorporating broker's latency, excluding slower brokers from sending data
   6. Once we have a nodes, their respective partitions and batches for each partition to send, we have to transform that data 
      into a format that can be sent to the broker - to the `ProduceRequest`. Kafka has its own binary [protocol](https://kafka.apache.org/protocol) 
      for that. The request to a single node looks more/less like this:
      ```
      ProduceRequest(
        transactionalId = 1, // for more details see my previous post about transactions 
        acks = 1, // see my post about replication and acks
        timeoutMs = 1000,
        topicData = [
          TopicData(
            name = "topic",
            partitionData = [
              PartitionData(
                index = 0,
                records = [
                  Record(
                    key = "key",
                    value = "value"
                    headers = [...]
                    ...
                  )
                ]
              )
            ]
          )
        ]
      )
      ```
      Obviously the real request is more complex, but this is the gist of it. Single producer instance can send multiple 
      requests like that to multiple brokers at the same time. 
   7. The response contains metadata about each batch for each topic partition. For each record in each batch KafkaProducer 
      then call user provided callback with that metadata as an argument. Note that the callback is called in the same thread 
      as the rest of producer's stuff, so don't use the blocking or long-running operations there. 
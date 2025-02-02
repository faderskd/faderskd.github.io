---
layout: post
title: "Kafka producer deep dive"
date: 2024-10-14
---

> [INFO]  
> The post covers replication KafkaProducer version (>=3.9).
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

We'll start with something really simple. This is the bare minimum configuration and the simplest usage. 
Don't get this tiny example seriously - this code sucks, but we'll improve it. 

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

### KafkaProducer send() 10000 foot view

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
serializer is easy - just implement the `org.apache.kafka.common.serialization.Serializer` interface. 
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
        in sending mostly empty batches. Additionaly, each batch consumes constant memory (`batch.size`) from a memory pool 
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
   3. And now is right time for more configuration options.
   4. [acks](https://kafka.apache.org/documentation/#producerconfigs_acks) - controls how many replicas must acknowledge 
   the record before the producer considers the record as sent. I'll stick with the DRY principle this time and really 
   encourage you to read my post about [Kafka replication]({% post_url 2024-05-15-kafka-replication %}). 
   It deeply explains how replication works in Kafka and how the `acks` affects the producer's behavior in different 
   scenarios.
   5. [delivery.timeout.ms](https://kafka.apache.org/documentation/#producerconfigs_delivery.timeout.ms) - it is the total 
   time for delivery of the record, after calling (and returning from) `send()` method. Look at the diagram below to see the 
   full message timeline. It contains time for batching, sending, waiting for response and retries.
   6. [max.request.size](https://kafka.apache.org/documentation/#producerconfigs_max.request.size) - as you saw previously, 
   the request contains a list of batches. Beside that there is some metadata attached to each request. 
   This configuration controls the maximum size of bytes send over the network to a single broker. The default value is 1MB.
   7. [retries](https://kafka.apache.org/documentation/#producerconfigs_retries) - the number of retries the producer will 
   make before giving up sending the request. This applies only to retryable errors like network issues, transient broker 
   problems, stale metadata, etc. You can achieve the same effect by disabling retries and handle each error on your own 
   when you get error from the `send()` future. In practice however, let leave its default value of `INTEGER.MAX_VALUE` and 
   configure just `delivery.timeout.ms`. It will stop sending after a configured timeout. 
   8. [request.timeout.ms](https://kafka.apache.org/documentation/#producerconfigs_request.timeout.ms) - how much time to wait 
   for a response from the broker. If the broker doesn't respond in that time, the producer will retry the request. 
   9. [max.block.ms](https://kafka.apache.org/documentation/#producerconfigs_max.block.ms) - before `send()` method 
   returns it has to have enough space for appending the record to the batch. If the current batch is full (or there is no batch
   for a specific partition) the producer has to allocate memory for a new batch. If there is no memory available (because 
   of `buffer.memory` limit), the producer will wait until another batch is sent and memory is freed. Another case when this method
   blocks is when sending to a topic which the producer has no metadata for. The producer will block until metadata is fetched.
   10. [enable.idempotence](https://kafka.apache.org/documentation/#producerconfigs_enable.idempotence) - config enabling 
   protection from duplicates in case of retries. The simple example of duplicated message is when a producer 
   sends message, the brokers processes it, saves it to the log, and just before returning the response to the producer, the producer times out 
   because of `request.timeout.ms`. The retry will happen but the broker already saved the message. This config prevents from 
   such cases. But it is not for free. It requires `acks=all` and `retries > 0` to be configured. The second it not a problem but
   `acks=all` is not always a case. By default, the idempotence is enabled, but only if there are no conflicting configs. So for 
   example if you configure `acks=1` the producer will disable idempotence. 
   For explanation how it internally works see **Idempotent Producer** chapter in [Kafka transactions post]({% post_url 2024-02-21-kafka-transactions %}).
   11. [max.in.flight.requests.per.connection](https://kafka.apache.org/documentation/#producerconfigs_max.in.flight.requests.per.connection) -
   how many requests can be sent to a broker without waiting for a response. The default value is 5. We'll see the effects 
   of this configuration later.     

And these are the most important configurations for the producer. Obviously there are many more, but they are more advanced 
and configured much less frequently. We will cover few of them when touching internals of the producer later.  

Because we've covered a lot of timeouts let's visualize the message timeline to see when each of them is used. 

 ![send-timeline.png]({{site.baseurl}}/img/producer/send-timeline.png)

### Parameters tuning

Kafka comes with a useful tool `kafka-producer-perf-test.sh` that can be used to measure the producer's throughput and latency. 
We can easily how different configurations affect the producer's performance. Let's check this out, and then we try to get similar 
results when running our own producer.  

Instructions setup Kafka cluster and CLI tool:
1. Clone my repository for setup of 3-node Kafka cluster [here](https://github.com/faderskd/kafka-playground). 
2. Run `docker-compose up`.
3. Download Kafka binaries https://kafka.apache.org/downloads (Kafka `3.8.1`).
4. Unpack and go to the `bin` directory.
5. Then create a topic with 3 partitions and replication factor 3
```postgresql
 ./kafka-topics.sh --create --topic userActivity --bootstrap-server localhost:9092 --replication-factor 3 --partitions 3
```
6. The resulting setup is
```postgresql
./kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic=userActivity
Topic: userActivity     TopicId: VXPhD3_zRE6WBMr1RI7NYg PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: userActivity     Partition: 0    Leader: 2       Replicas: 2,3,4 Isr: 2,3,4      Elr:    LastKnownElr: 
        Topic: userActivity     Partition: 1    Leader: 3       Replicas: 3,4,1 Isr: 3,4,1      Elr:    LastKnownElr: 
        Topic: userActivity     Partition: 2    Leader: 4       Replicas: 4,1,2 Isr: 4,1,2      Elr:    LastKnownElr: 
```
7. I'm additionally using [toxiproxy](https://github.com/Shopify/toxiproxy?tab=readme-ov-file#1-installing-toxiproxy) 
for simulating network latency as I'm running everything on my local machine and network latency is just unrealistically low. 
In the repository there is a `toxiproxy.json` file defining proxy configuration for kafka brokers. So start a server in one terminal.
```postgresql
toxiproxy-server -config toxiproxy.json
```
7. Add latency for the brokers. This will add 100[+/-50]ms of latency to each broker. The same setup for brokers and 
latency is used in this and all further examples.
```postgresql
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka1
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka2
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka3
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka4
```

Now we can run the producer performance test. Let's say that we have an application that is not so much sensitive to delays.
We want to send as many messages as possible, but the maximum time from message being passed to the producer, to the delivery 
should not exceed 10s.

* `--record-size 200` - set the record size to 200 bytes
* `--num-records 20000000` - send 20M records
* `--throughput=-1` - don't limit the throughput - we want to get the highest possible throughput
* `--bootstrap.servers=localhost:9092` - for bootstrap servers use one of the brokers, the rest will be discovered automatically
* `--delivery.timeout.ms=10000` - limit the maximum time for delivery up to 10s
* `--request.timeout.ms=3000` - when exceeding the request timeout the producer will retry the request
* `--linger.ms=0` - don't wait for batch to fill and send whatever there is. Note that the producer may still batch some of the 
messages if the producing throughput is high. The producer is doing other things in the meantime when we add new messages 
It will just not wait any additional time for the batch to fill 
* `--batch.size=16384` (2^14) - max batch size 16KB
* `--acks=1` - wait only for the leader to acknowledge the record
* `--print-metrics` - print metrics to the console after the test is finished (there will be a lot of them)

```postgresql
./kafka-producer-perf-test.sh --record-size 200 --num-records 20000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 delivery.timeout.ms=10000 request.timeout.ms=3000 batch.size=16384 acks=1 \
  --print-metrics

org.apache.kafka.common.errors.TimeoutException: Expiring 78 record(s) for userActivity-1:10001 ms has passed since batch creation
```

Hmm, something gone wrong... We are hitting the 10s threshold for a delivery time. Let's temporarily remove any time limits and see
what the producer can achieve without them. 

```postgresql
./kafka-producer-perf-test.sh --record-size 200 --num-records 20000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 batch.size=16384 acks=1 --print-metrics

48282 records sent, 9551.3 records/sec (1.82 MB/sec), 12068.7 ms avg latency, 14608.0 ms max latency.
48043 records sent, 9599.0 records/sec (1.83 MB/sec), 16159.7 ms avg latency, 16793.0 ms max latency.
48194 records sent, 9610.0 records/sec (1.83 MB/sec), 16615.7 ms avg latency, 16895.0 ms max latency.
...
```

Well, I had to decrease a number of total messages sent as I would like to see metrics at the end of the experiment and with the 
current speed of ~`9500 records/sec` it would take 35~ minutes. Let's temporarily decrease it to 2M messages.

The experiment took few minutes, and we got the following results (I cut out the most of currently not needed metrics):

```postgresql
./kafka-producer-perf-test.sh --record-size 200 --num-records 2000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 batch.size=16384 acks=1 --print-metrics
...
49294 records sent, 9850.9 records/sec (1.88 MB/sec), 16296.2 ms avg latency, 16736.0 ms max latency.
49530 records sent, 9854.8 records/sec (1.88 MB/sec), 16293.3 ms avg latency, 16671.0 ms max latency.
49139 records sent, 9796.5 records/sec (1.87 MB/sec), 16241.8 ms avg latency, 16617.0 ms max latency.
2000000 records sent, 9839.372248 records/sec (1.88 MB/sec), 15532.49 ms avg latency, 16736.00 ms max latency, 16146 ms 50th, 16520 ms 95th, 16614 ms 99th, 16692 ms 99.9th.

Metric Name                                                                                      Value
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                               : 16373.882
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                        : 16111.299
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                          : 119.003
producer-metrics:request-rate:{client-id=perf-producer-client}                                 : 125.643
producer-metrics:request-size-avg:{client-id=perf-producer-client}                             : 16442.260
producer-topic-metrics:record-send-rate:{client-id=perf-producer-client, topic=userActivity}   : 9795.937
...
```

Interpretation:
1. Firstly we see that 99th percentile for delivery time is ~26s. So the 99% of requests are delivered in less than 26s. 
It is higher than what we wanted (10s).
2. Looking at the batch size `batch-size-avg` we can see that the average value is ~16KB. This is the maximum batch size we set. So 
the batches are sent full. 
3. We set `linger.ms` to 0 so the producer is not waiting for the batch to fill. But still, the producer is batching. 
The pace of records coming to the producer is higher that it can actually send. 
4. The queuing time `record-queue-time-avg=16111` indicates that the batches are spending about 16s in the queue before 
being sent. This is quite a lot. This is also very close to the 99th percentile of the latency.
5. Note also that the request avg size `request-size-avg` is almost as batch size `batch-size-avg`. This indicates that
requests to a brokers contains just one batch. This is expected as we are sending messages to one topic, and each partition 
of that topic is on a separated broker.
6. The latency for a single request is ~119ms. We can send the `1000 / 119 = 8.4` requests per second to a broker. Additionally,
we have `max.in.flight.requests.per.connection=5` (default value) so we can send 5 requests upfront without waiting for response. 
This gives us `8.4 * 5 = 42` requests per second to a single broker. We have 3 brokers so the total number of requests is `42 * 3 = 126`. 
And it is very close to the metric reported as `request-rate=125.643`.
7. The cause of high delivery time is queuing. The broker response times are not that high. The queuing happens because we 
are not sending incoming records fast enough. The formula for the number of records we can currently send is like:
 
```postgresql
records-per-second = request-rate * (request-size-avg - request-overhead) / record-size 
```

The request overhead is any data not related to the record itself. We can assume it is negligible comparing to the whole 
request size. We've set record size to 200 bytes, so 

```postgresql
records-per-second = request-rate (125) * request-size-avg (16442) / record-size (200)` =>
records-per-second =~ 10K
```

and this is close to what we see in `record-send-rate=9795` metric. Going further with the calculation, we know that 
the request size is almost the same as batch size:

```postgresql
records-per-second = request-rate * request-size-avg / record_size 
records-per-second = request-rate * batch_size-avg / record_size 
```

we also know the formula for `request-rate` so we can substitute it to the above equation:

```postgresql
records-per-second = request-rate * batch_size-avg / record_size
records-per-second = (1000 / request-latency-avg) * batch_size-avg / record_size
```

To increase the number of records sent, and get rid of queuing we can: decrease the request latency or increase the batch 
size. Decreasing latency would be much harder (as it requires tuning java, os/network or brokers) than just increasing batch_size, 
and we'll try that. Note that increasing batch size may cause increasing latency as we will start sending more data 
to process in a single request. But let's do this and see what happens. I'll double the batch size to 32KB. 

```postgresql
./kafka-producer-perf-test.sh --record-size 200 --num-records 2000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 batch.size=32768 acks=1 --print-metrics

...
98903 records sent, 19764.8 records/sec (3.77 MB/sec), 8076.9 ms avg latency, 8308.0 ms max latency.
97654 records sent, 19515.2 records/sec (3.72 MB/sec), 8087.9 ms avg latency, 8437.0 ms max latency.
2000000 records sent, 19583.843329 records/sec (3.74 MB/sec), 7788.41 ms avg latency, 8480.00 ms max latency, 8092 ms 50th, 8306 ms 95th, 8368 ms 99th, 8426 ms 99.9th.

Metric Name                                                                                      Value
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                               : 32755.845
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                        : 7975.961
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                          : 118.658
producer-metrics:request-rate:{client-id=perf-producer-client}                                 : 125.901
producer-metrics:request-size-avg:{client-id=perf-producer-client}                             : 32822.848
producer-topic-metrics:record-send-rate:{client-id=perf-producer-client, topic=userActivity}   : 19645.206
```

As you see despite increasing the batch size the request latency didn't change. The queuing time is also lower. The rate of 
records send (throughput) is higher. We can go further and try to increase the batch size even more. Let's set it to 
2^18 = 262144 bytes. What's more I'll restore previous delivery/request timeout because our initial requirement was 
that message should be published up to 10s. I'll also send 20M messages as in the first failed try. 

```postgresql
./kafka-producer-perf-test.sh --record-size 200 --num-records 20000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 batch.size=262144 acks=1 delivery.timeout.ms=10000 request.timeout.ms=3000 \
  --print-metrics

...
755040 records sent, 150887.3 records/sec (28.78 MB/sec), 1036.3 ms avg latency, 1225.0 ms max latency.
762788 records sent, 152557.6 records/sec (29.10 MB/sec), 1038.1 ms avg latency, 1241.0 ms max latency.
20000000 records sent, 153622.809915 records/sec (29.30 MB/sec), 1015.29 ms avg latency, 1263.00 ms max latency, 1017 ms 50th, 1127 ms 95th, 1168 ms 99th, 1225 ms 99.9th.

Metric Name                                                                                      Value
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                               : 262066.102
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                        : 899.916
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                          : 121.223
producer-metrics:request-rate:{client-id=perf-producer-client}                                 : 123.197
producer-metrics:request-size-avg:{client-id=perf-producer-client}                             : 262133.141
producer-topic-metrics:record-send-rate:{client-id=perf-producer-client, topic=userActivity}   : 153671.628
```

Now it completed within 2 minutes. Look at this impressive throughput of 150K records/sec. 99% of the messages are delivered 
below 1.2s. What is interesting, the avg request latency almost didn't change. And there is still a potential to improve 
the throughput because the batch is full. But let's stop here and do a little summary of the experiment.

#### Kafka performance tool summary
1. This example is not a generic way of tuning the producer. We could do some assumptions specific to our case, e.g. that 
a request size is almost equal to the batch size. When you would send messages to multiple topics with more partitions 
it would look like differently. The goal was to show the approach to tuning the producer. 
2. We started with defining performance requirements.
3. Then we used the producer performance tool to see what we can achieve with some basic configuration. 
4. Then we checked metrics, saw if the requirements are met, and tried to improve by tweaking the configuration. 
5. Then we've iterated until we get sufficient performance.  

By the end of this post we'll cover more advanced ways of improving the producer's performance. But we'll need to 
understand much more about its internals :)

### Kafka producer improved usage

It's time to improve our first producer code sample. We'll add some more configurations, and we'll try to make it more 
robust by using async sending. The goal is to get similar results as in performance test tool:

```postgresql
20000000 records sent, 153622.809915 records/sec (29.30 MB/sec), 1015.29 ms avg latency, 1263.00 ms max latency, 1017 ms 50th, 1127 ms 95th, 1168 ms 99th, 1225 ms 99.9th.
```
* sending 20M records
* each record size is 200 bytes
* with throughput ~150K records/sec
* with p99 delivery latency ~1.2s

```java
public class AsyncAuditProducer implements ProgramLoop {
    private final static Logger logger = LoggerFactory.getLogger(AsyncAuditProducer.class);
    private final static Random random = new Random();
    private final static String AUDIT_TOPIC = "userActivity";

    private volatile boolean running = true;
    private final AtomicInteger sentCounter = new AtomicInteger(0);
    private final Timer timer;
    private final KafkaProducer<byte[], byte[]> producer;
    private long lastReportMillis = System.currentTimeMillis();

    public AsyncAuditProducer() {
        this.producer = new KafkaProducer<>(producerProperties());
        MeterRegistry meterRegistry = new SimpleMeterRegistry();
        timer = Timer
                .builder("send.latency")
                .publishPercentiles(0.99)
                .register(meterRegistry);
    }

    private static Properties producerProperties() {
        Properties props = new Properties();
        // required parameters
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9292,localhost:9393,localhost:9494,localhost:9595");
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());

        // batching
        props.setProperty(ProducerConfig.LINGER_MS_CONFIG, "0");
        props.setProperty(ProducerConfig.BATCH_SIZE_CONFIG, "262144"); // 256KB

        // sending
        props.setProperty(ProducerConfig.ACKS_CONFIG, "1"); // only leader confirms
        props.setProperty(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, "10000"); // 10s
        props.setProperty(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, "3000"); // 3s
        props.setProperty(ProducerConfig.MAX_BLOCK_MS_CONFIG, "1000"); // 1s

        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    byte[] payload = generatePayload(200);
                    ProducerRecord<byte[], byte[]> record =
                            new ProducerRecord<>(AUDIT_TOPIC, payload);

                    long sendTime = System.currentTimeMillis();

                    producer.send(record, (metadata, exception) -> {
                        if (exception != null) {
                            logger.error("Error while sending audit event", exception);
                        } else {
                            timer.record(System.currentTimeMillis() - sendTime, TimeUnit.MILLISECONDS);
                            sentCounter.incrementAndGet();
                            reportMetrics(startMillis);
                        }
                    });
                    if (sentCounter.get() >= 20000000) {
                        break;
                    }
                } catch (Exception ex) {
                    logger.error("Error while sending audit event", ex);
                }
            }
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }

    private void reportMetrics(long startMillis) {
        // report metrics every 3s
        if (System.currentTimeMillis() - lastReportMillis > 3000) {
            lastReportMillis = System.currentTimeMillis();
            double throughput = 1000 * ((double) sentCounter.get() / (System.currentTimeMillis() - startMillis));
            logger.info("------------------------- Reporting metrics --------------------------------------");
            String throughputMsg = String.format("Send %dK messages. Throughput: %.2f records/s", sentCounter.get() / 1000, throughput);
            logger.info(throughputMsg);
            Arrays.stream(timer.takeSnapshot().percentileValues()).forEach(
                    percentile -> logger.info(
                            "Percentile {} : {}", percentile.percentile(), percentile.value(TimeUnit.MILLISECONDS))
            );
        }
    }

    @Override
    public void wakeup() {
        logger.info("Program loop wakeup");
        running = false;
    }

    private static byte[] generatePayload(int length) {
        byte[] payload = new byte[length];
        for (int i = 0; i < length; i++) {
            // ASCII A-Z
            payload[i] = (byte) (random.nextInt(26) + 65);
        }
        return payload;
    }
}
```

Results:
```postgresql
...
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 18436K messages. Throughput: 153268.10 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 633.339904

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 18872K messages. Throughput: 153070.52 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 601.882624

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19267K messages. Throughput: 152553.26 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 601.882624

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19716K messages. Throughput: 152485.55 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 601.882624
```

We've managed to get similar results as in the performance test tool. Note that the measured latency is actually 
the e2e latency from calling `send()` to executing provided callback. In the performance test tool, the latency was measured the 
same way. This time however we got event better p99 results. 

We will try to do another round of optimizations later when talking more about internals. Let's add some more features to
our producer.

### Adding serialization
The first thing is that in the producer example we are generating payload bytes ready to sent without any serialization. 
The `ByteArraySerializer` does nothing with the payload (it returns the same payload as given in argument . This is the 
same what the kafka performance tool does. More realistic case would be to use some form of serialization like 
[Avro](https://avro.apache.org/docs/), any other binary format or just JSON. We'll use serialization to JSON now.
I've added serialization before sending the record. The same result could be achieved by implementing custom kafka 
producer serializer. 

```java
public class AsyncAuditProducer implements ProgramLoop {
    ...
    private static Properties producerProperties() {
        Properties props = new Properties();
        ...
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        ...
        
        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    AuditLog auditLog = generateExampleAuditLog();

                    ProducerRecord<byte[], byte[]> record =
                            new ProducerRecord<>(AUDIT_TOPIC, mapper.writeValueAsBytes(auditLog));

                    long sendTime = System.currentTimeMillis();

                    ...
                } catch (Exception ex) {
                    logger.error("Error while sending audit event", ex);
                }
            }
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }
    
    // after JSON serialization it takes ~150 bytes
    private static AuditLog generateExampleAuditLog() {
        ActionType actionType = ActionType.values()[(int) (Math.random() * ActionType.values().length)];
        String userId = "user-" + UUID.randomUUID();
        long timestamp = Instant.now().toEpochMilli();
        return new AuditLog(timestamp, userId, actionType, UUID.randomUUID().toString());
    }
}
```

Our serialized `AuditLog` object is around 150 bytes now. The time for generating example payload changed too, so comparing 
the results with previous ones is not fair. Let's just see what throughput/latency we can get with our current settings. 

```postgresql
...
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19577K messages. Throughput: 191498.05 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1174.40512

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 20148K messages. Throughput: 191440.84 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1174.40512
```

### Adding metadata in record headers
The next common pattern in producer usage is to add metadata associated with the record. You can put here any information 
which don't fit well into the payload, e.g. routing, tracing. The benefit is also that you don't have to deserialize the 
payload on the consumer side to get this metadata. We'll add a simple tracing information to the record headers.

```java
public class AsyncAuditProducer implements ProgramLoop {
    ...
    private static Properties producerProperties() {
        Properties props = new Properties();
        ...
        
        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    AuditLog auditLog = generateExampleAuditLog();

                    ProducerRecord<byte[], byte[]> record =
                            new ProducerRecord<>(AUDIT_TOPIC, mapper.writeValueAsBytes(auditLog));
                    // Add tracing information
                    byte [] traceId = "SomeTraceIdFromUpperLayers".getBytes(StandardCharsets.UTF_8);
                    record.headers().add("trace-id", traceId);
                    
                    ...
                } catch (Exception ex) {
                    logger.error("Error while sending audit event", ex);
                }
            }
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }
    ...
}
```

Performance results with settings untouched and added header:

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19341K messages. Throughput: 156832.18 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1174.40512

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19828K messages. Throughput: 156960.12 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1174.40512
```

The throughput decreased. Obviously headers take some space so fewer data can be packed into a batch. Remember that 
we didn't optimize the producer as best we could. We will do this later. I'm just showing you that headers, as any added 
data don't come for free.

### Partitioning with key
As mentioned at the beginning of the post, we can choose a partitioning for the record in a few ways. Out of the box solution 
is that the producer picks a partition based on its internal load balancing algorithm (more about it later). This happened 
so far in previous examples because we didn't configured anything for partitioning, and didn't use key.  

Given a record key, the producer calculates a hash of the key and uses that to pick a partition. Let's try to route all 
records to the same partition. Our throughput should decrease ~3 times because we have 3 partitions, and now everything 
goes to one of them. We are effectively decreasing a parallelism (more about it later). 

```java
public class AsyncAuditProducer implements ProgramLoop {
    ...

    private static Properties producerProperties() {
        Properties props = new Properties();
        ...

        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    AuditLog auditLog = generateExampleAuditLog();
                    
                    // all records will go to the same partition: hash(someKey) % numberOfPartitions
                    byte[] someKey = "AuditLogGenericKey".getBytes(StandardCharsets.UTF_8);
                    ProducerRecord<byte[], byte[]> record =
                            // add key to the record
                            new ProducerRecord<>(AUDIT_TOPIC, someKey, mapper.writeValueAsBytes(auditLog));

                    byte [] traceId = "SomeTraceIdFromUpperLayers".getBytes(StandardCharsets.UTF_8);
                    record.headers().add("trace-id", traceId);
                    ...
            }
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }
    ...
}
```

The results in the previous benchmark with headers was ~150K records/sec. Now it is:

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19832K messages. Throughput: 47707.61 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 3355.4432

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19978K messages. Throughput: 47708.89 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 3355.4432
```

The throughput decreased as expected to ~50K. The latency increased probably because the queuing started to take effect. To be 
sure I'd have to check producer metrics, but I'll describe it later. Alternatively we could use the kafka performance tool 
with similar config and see metrics there. 

So why one may even need that routing? One of the reasons is that you may want to preserve order of the produced records. 
Once they go to the same partition, they will be processed in the order they were sent. For this to work we will need to 
reassure that another few producer configs are set correctly (descried in dedicated section later). Consuming from multiple
Kafka partitions can happen in parallel by independent processes. So once you sent specific records to the same partition, you're 
forcing the consumer to process them in the order they were sent. By specifying the key, messages with the same key
(thus with the same hash) will go to the same partition.  

### Partitioning with custom partitioner

If you want to apply custom partitioning strategy you can implement your own partitioner. The very simple one would be 
e.g. round-robin partitioner. Note that the partitioner should be thread-safe as it is called by 
`send()` method which can be called by multiple threads. Our example is just using single calling thread. 

```java
// AsyncAuditProducer.java
public class AsyncAuditProducer implements ProgramLoop {
    ...
    private static Properties producerProperties() {
        Properties props = new Properties();
        ...
        // partitioning
        props.setProperty(ProducerConfig.PARTITIONER_CLASS_CONFIG, RoundRobinPartitioner.class.getName());
        // custom property used in our round robin partitioner
        props.setProperty("monitoring.enabled", "false");
        ...
        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    AuditLog auditLog = generateExampleAuditLog();

                    ProducerRecord<byte[], byte[]> record =
                            new ProducerRecord<>(AUDIT_TOPIC, mapper.writeValueAsBytes(auditLog));

                    byte [] traceId = "SomeTraceIdFromUpperLayers".getBytes(StandardCharsets.UTF_8);
                    record.headers().add("trace-id", traceId);
                    ...
                }
            } finally {
                producer.close();
                logger.info("Closing producer...");
            }
        }
    ...
}

// RoundRobinPartitioner.java
public class RoundRobinPartitioner implements Partitioner {
    private static final Logger logger = LoggerFactory.getLogger(RoundRobinPartitioner.class);
    private ConcurrentHashMap<String, AtomicInteger> topicsCounters = new ConcurrentHashMap<>();
    private boolean monitoringEnabled;

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        AtomicInteger topicCounter = topicsCounters.computeIfAbsent(topic, (t) -> new AtomicInteger());
        if (monitoringEnabled) {
            // measure something
        }
        return topicCounter.incrementAndGet() % cluster.partitionCountForTopic(topic);
    }

    @Override
    public void close() {
    }

    @Override
    public void configure(Map<String, ?> configs) {
        // getting custom properties
        monitoringEnabled = Boolean.parseBoolean((String) configs.get("monitoring.enabled"));
        logger.info("Staring custom round robin partitioner with monitoring enabled: {}", monitoringEnabled);
    }
}
```

The results are similar to the one with built-in partitioner from previous examples without a key.

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19325K messages. Throughput: 156768.86 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1539.309568

producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19794K messages. Throughput: 156750.33 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1539.309568
```

The throughput is ~157K, p99 is ~1.5s. Note that there is already similar ready-to-use implementation in kafka clients 
package: `org.apache.kafka.clients.producer.RoundRobinPartitioner`. Implementing a custom partitioner is rather less 
common, and it is good to know what out of the box solutions are available. 

### Kafka producer sending internals

Before describing the default partitioner strategy, I have to cover a little more about the producer internals. Kafka producer
sending process involves few components. The diagram below shows how each component is used by the other.

// P3 excalidraw

The process of sending a message comes down to (with components in square brackets where the things happen):  

1. [**`KafkaProducer`**] is a main entry point for sending records with a `send(ProducerRecord)` method. 
2. [**`ProducerMetadata`**] The first thing the producer is doing is searching for metadata of the record's topic in metadata cache. If it is not 
available, the producer will schedule an update and [wait](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1027).
Metadata is information about the topic like partitions, replicas, leaders, etc. 
3. Once it has metadata, the serialization [happens](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1046) as mentioned before.
4. [**`Partitioner (custom)`**] - if the partition was not given in the record, [use custom](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1463)) partitioner if present. This is 
the interface we've implemented with `RoundRobinPartitioner`.
5. [**`Partitioner (built-in)`**] - if partition is not resolved **and record has key**, [use murmur2 hash](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1475) 
from Kafka built-in partitioner to calculate partition.
6. [**`RecordAccumulator`**] - finally, [append the record](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1071) to the accumulator. 
If partition was still not resolved, it will [happen](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L310) 
in the accumulator using built-in partitioner. The selection process depends on different partitioner configurations, but we have 
few choices here:
    - based on the performance metrics (adaptive partitioning)
    - uniform distribution - each partition gets equal number of bytes (adaptive partitioning disabled)
7. [**`RecordAccumulator - append to batch`**] - append the record to the existing batch or create a new one if the current batch is closed - 
because of the size or linger time. For each of the topic partition the accumulator maintains a [queue](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L325) 
of batches. Some of them are ready to sent, because they are full or linger time passed. 
// P4 image with queues in accumulator
8. [**`RecordAccumulator - allocate new batch`**] - if the latest batch for a partition was closed because it is full or linger time passed and 
the sending started, the accumulator will try to allocate a new one. 
9. [**`BufferPool`**] - the responsibility of keeping the memory for the batches is on the buffer pool. If the record accumulator 
tries to allocate a batch without sufficient memory we will have to [wait](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/BufferPool.java#L153).
10. When all of the above succeeded, the producer returns a [future](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1099) 
to the `send()` method caller, which completes when the record is sent.
11. [**`Sender - io thread`**] - the producer has a single [IO thread](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L467) 
running in background sender's [loop](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java#L249). 
Each iteration of the loop will try to find and send ready brokers nodes, with partitions having batches ready to send. 
  - Broker readiness is determined based on its connection state: if it is connected, not backing off, and number of in-flight
    doesn't exceed the `max.in.flight.requests.per.connection` limit.
  - Batches [readiness](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L625) 
    is mostly determined based on batch size and linger. There are some exceptions, e.g. when the buffer pool is full the sender 
    will try to send the batch immediately to free some memory. The other can be problems with partitions leaders, etc.  
12. [**`Sender - draining batches`**] - once the sender found batches to send it starts [draining](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L869) 
them from respective partitions' queues. The current implementation (3.9) drains only a [single](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L915)
ready batch for each partition in a single loop run.
// p5 image with draining batches stage 1
// p6 image with draining batches stage 2
13. [**`Sender - sending to broker`**] - drained batches are put into `ProduceRequest` and [sent](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java#L451)
to the broker. The request has its own [binary format](https://kafka.apache.org/protocol#The_Messages_Produce), but it's 
current structure looks like this:

```java
ProduceRequest(
        transactionalId = 1, // for more details see my previous post about transactions
        acks = 1, // see my post about replication and acks
        timeoutMs = 1000, // info for the broker how long the client is willing to wait for the response
        topicData = [
            TopicData(
                name = "topic",
                partitionData = [
                    PartitionData(
                        index = 0, // partition number
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
// p7 image with sending to broker

And that sending can happen concurrently for multiple requests on a single network thread, because of the usage of 
selector API from Java NIO. The networking stuff is [here,](https://github.com/apache/kafka/blob/3.9/clients/src/main/java/org/apache/kafka/clients/NetworkClient.java#L568)
and it is a big piece of low-level code.

### Built-in partitioner
Let's go back to the built-in partitioner. Now we can understand what it takes into account when making a decision about 
partitioning. 
Recall that if we don't specify the record's partition used with `send(ProducerRecord)`, don't supply custom partitioner class, and don't
provide a key, the producer will use its built-in partitioner. And this is fired [here](https://github.com/apache/kafka/blob/409a43eff77511e89bba2f95934cb1ebc417236d/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L310) 
in the producer code. It is a so-called adaptive sticky partitioner, and it has quite sophisticated algorithm for 
data distribution. But to understand its advantages, we'll start by comparing it with our round-robin partitioner. The results for 
tests run with built-in partitioner for example [while testing headers](#adding-metadata-in-record-headers) 
and tests with [custom partitioner](#partitioning-with-custom-partitioner) didn't differ so much in terms of throughput and latency.  

Built-in partitioner:  
```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19828K messages. Throughput: 156960.12 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1174.40512
```

Round-robin partitioner:  

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 19794K messages. Throughput: 156750.33 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 1539.309568
```

But what happens when one of the broker is slow? To find out, I've added additional latency to one of the brokers. So now 
broker `kafka1` has `300ms` of added latency while others have `100ms` as previously:

```postgresql
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=300 -a jitter=50 kafka1
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka2
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka3
toxiproxy-cli toxic add -t latency -n kafkaToxic -a latency=100 -a jitter=50 kafka4
```

```postgresql
~/D/k/bin ❯❯❯ ./kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic=userActivity

Topic: userActivity     TopicId: 5Jy7kkJIRQ-4oJ--YgekVA PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: userActivity     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: userActivity     Partition: 1    Leader: 2       Replicas: 2,3,4 Isr: 2,3,4
        Topic: userActivity     Partition: 2    Leader: 3       Replicas: 3,4,1 Isr: 3,4,1
```

`Kafka1` broker is seen as the node `1` in the output above so it is a leader for partition `0`. 

Round-robin partitioner (I sent only 10M messages because of impatience):

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 10118K messages. Throughput: 44813.40 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 7782.531072
```

Built-in, adaptive sticky partitioner:  

```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 10880K messages. Throughput: 124688.25 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 3053.453312
```

Built-in partitioner is much more resilient to slow brokers. The throughput is still high, the latency increased compared
to tests with normal latency. It seems like the built-in partitioner is more aware of the broker's state and can adapt. 

#### Built-in partitioner - more metrics

Ok, what about measuring the number of records sent to each partition? We can do this by adding some monitoring to the code. 
We'll use `partitionsSendCounters` map with atomic counter for each partition. 

```java
public class AsyncAuditProducer implements ProgramLoop {
    ...
    private final AtomicInteger sentCounter = new AtomicInteger(0);
    private final ConcurrentHashMap<Integer, AtomicInteger> partitionsSendCounters = new ConcurrentHashMap<>();

    private static Properties producerProperties() {
        ...
        // partitioning - remove this line to use built-in partitioner
        props.setProperty(ProducerConfig.PARTITIONER_CLASS_CONFIG, RoundRobinPartitioner.class.getName());
        ...

        return props;
    }

    @Override
    public void start() {
        try {
            long startMillis = System.currentTimeMillis();
            while (running) {
                try {
                    ...
                    producer.send(record, (metadata, exception) -> {
                        if (exception != null) {
                            logger.error("Error while sending audit event", exception);
                        } else {
                            timer.record(System.currentTimeMillis() - sendTime, TimeUnit.MILLISECONDS);
                            sentCounter.incrementAndGet();
                            // add metrics per partition
                            partitionsSendCounters.
                                    computeIfAbsent(metadata.partition(), (p) -> new AtomicInteger())
                                    .incrementAndGet();
                            reportMetrics(startMillis);
                        }
                    });
                    ...
        } finally {
            producer.close();
            logger.info("Closing producer...");
        }
    }

    private void reportMetrics(long startMillis) {
        if (System.currentTimeMillis() - lastReportMillis > 3000) {
            ...
            partitionsSendCounters.forEach((partition, counter) ->
                    logger.info("Partition {} : {}", partition, counter.get()));
        }
    }
}
```

The results for round-robin partitioner:  
```postgresql
producer.AsyncAuditProducer - ------------------------- Reporting metrics --------------------------------------
producer.AsyncAuditProducer - Send 20105K messages. Throughput: 43870.86 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 8050.966528
producer.AsyncAuditProducer - Partition 0 : 8059532
producer.AsyncAuditProducer - Partition 1 : 9249785
producer.AsyncAuditProducer - Partition 2 : 2796126
```

Built-in partitioner:  
```postgresql
producer.AsyncAuditProducer - Send 20017K messages. Throughput: 123201.39 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 2919.235584
producer.AsyncAuditProducer - Partition 0 : 3280836
producer.AsyncAuditProducer - Partition 1 : 8344435
producer.AsyncAuditProducer - Partition 2 : 8391752
```

The round-robin partitioner sent less record to a healthy partition `2` while almost equally to partitions `0` (unhealthy) and `1`.
The built-in partitioner on the other hand, made a good job by giving a slow broker chance to catch his breath. But how it is 
possible that we call `topicCounter.incrementAndGet() % cluster.partitionCountForTopic(topic)` once for each message 
in our round-robin partitioner and the records are not evenly distributed? The answer is hidden in the implementation details but covering 
it would be too much for an already very long post. Spoiler: `partition(...)` of the partitioner is not necessarily called
once for each message, thus can produce skewed distribution when unequal latency happens. 

#### Built-in partitioner - more configuration

As you see the [built-in](https://cwiki.apache.org/confluence/display/KAFKA/KIP-794%3A+Strictly+Uniform+Sticky+Partitioner) partitioner is quite smart. 
What's interesting, previously in Kafka there was a [StickyPartitioner](https://cwiki.apache.org/confluence/display/KAFKA/KIP-480%3A+Sticky+Partitioner) which had a problem 
with skewed distribution too: slower partitions got more records. The [built-in adaptive partitioner](https://cwiki.apache.org/confluence/display/KAFKA/KIP-794%3A+Strictly+Uniform+Sticky+Partitioner)
fixes that problem. As stated before, if you don't specify the partitioner class, the built-in adaptive one is used. And we have a 
few configuration options to adjust its behavior when bad things happen.

**[partitioner.adaptive.partitioning.enable](https://kafka.apache.org/documentation/#producerconfigs_partitioner.adaptive.partitioning.enable)** - 
whether to adapt to broker performance. It is enabled by default, and we used in tests for built-in partitioner. When enabled 
the partitioner will:
1. check the queue sizes of the partitions


If we disable it the partitioner will:
1. pick a random partition
2. produce `batch.size` bytes of records
3. then switch to the next partition

```java
    private static Properties producerProperties() {
    Properties props = new Properties();
    ...
    // partitioning
    // disable our custom partitioner so the built-in will be used
    // props.setProperty(ProducerConfig.PARTITIONER_CLASS_CONFIG, RoundRobinPartitioner.class.getName());
    props.setProperty(ProducerConfig.PARTITIONER_ADPATIVE_PARTITIONING_ENABLE_CONFIG, "false");
    ...
    return props;
}
```

```postgresql
producer.AsyncAuditProducer - Send 20121K messages. Throughput: 52097.48 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 8050.966528
producer.AsyncAuditProducer - Partition 0 : 7746897
producer.AsyncAuditProducer - Partition 1 : 6079592
producer.AsyncAuditProducer - Partition 2 : 6295381
```

The latency came back to the round-robin partitioner level. The throughput is also similar. While the distribution is not perfect 
and the slow partitions got more messages it is much better than in round-robin partitioner. Messages generated by my `AsyncAuditProducer`
producer are roughly the same size, so the results indicate that the slowest broker indeed got more data even in case of built-in implementation.  
The diff between the slowest partition `0` and the partition `1` with the lowest number of messages is `7746897 - 6295381 =~ 1.4M`.
The throughput is `~52K` records/sec so the diff had to accrue over the longer period. In case of the round-robin partitioner 
the diff was `8059532 - 2796126 =~ 5.3M`.

I had to double-check and compare it with kafka-performance tool. Maybe I missed something in the producer code.  
The config and message size are the same as for our own producer benchmark above. The throughput may differ a little bit because 
we have added serialization, headers, etc. I just wanted to see if partitions will be equally loaded: 

```postgresql
./kafka-producer-perf-test.sh --record-size 150 --num-records 20000000 --throughput=-1 --topic userActivity \
  --producer-props bootstrap.servers=localhost:9092 delivery.timeout.ms=10000 request.timeout.ms=5000 batch.size=262144 acks=1 max.block.ms=1000 \
  --print-metrics

20000000 records sent, 69960.402412 records/sec (10.01 MB/sec), 2853.64 ms avg latency, 8330.00 ms max latency, 210 ms 50th, 7759 ms 95th, 7946 ms 99th, 8132 ms 99.9th.
...
producer-node-metrics:outgoing-byte-total:{client-id=perf-producer-client, node-id=node-1}     : 1182270704.000
producer-node-metrics:outgoing-byte-total:{client-id=perf-producer-client, node-id=node-2}     : 974298329.000
producer-node-metrics:outgoing-byte-total:{client-id=perf-producer-client, node-id=node-3}     : 1045194709.000
...
```

The diff between the slowest node (partition `0`) and node with the lowest number of bytes sent (partition `2`) is 
`1182270704 - 974298329 =~ 208MB`. The throughput is `~10MB/sec` so the diff had to accumulate over `~20s` and is 
probably not just the effect of unfortunate timing. So now we know that the adaptive partitioning is something we really need.
The `props.setProperty(ProducerConfig.PARTITIONER_ADPATIVE_PARTITIONING_ENABLE_CONFIG` should be left as `true`. 

**[partitioner.availability.timeout.ms](https://kafka.apache.org/documentation/#producerconfigs_partitioner.availability.timeout.ms)** - 
it is a time threshold the built-in partitioner waits for the ready batches to be started sending. Technically it means that:
1. if we define `readyTimestamp` as time when partition has ready to sent batches (`linger.ms` passed or batch is full)
2. and `drainTimestamp` as time when ready batches (waiting in a queue to be sent) are actually got from that queue, put into
produce request and sent to the broker. This happens if the producer has active connection with the broker, the max in-flight
messages were not reached, etc.
3. then if `readyTimestamp` - `drainTimestamp` > `partitioner.availability.timeout.ms` the partitioner will temporarily stop sending 
data to that partition because it is considered as too slow and causes higher latencies, queuing, etc.  

Let's try setting `partitioner.availability.timeout.ms` to `50ms`: 

```java
    private static Properties producerProperties() {
    Properties props = new Properties();
    ...
    // partitioning
    props.setProperty(ProducerConfig.PARTITIONER_ADPATIVE_PARTITIONING_ENABLE_CONFIG, "true");
    props.setProperty(ProducerConfig.PARTITIONER_AVAILABILITY_TIMEOUT_MS_CONFIG, "50");
    ...
    return props;
}
```

The results:
```postgresql
producer.AsyncAuditProducer - Send 19811K messages. Throughput: 120023.17 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 2682.257408
producer.AsyncAuditProducer - Partition 0 : 3328407
producer.AsyncAuditProducer - Partition 1 : 8178502
producer.AsyncAuditProducer - Partition 2 : 8304836
```

The results for adaptive partitioning without availability timeout were:

```postgresql
producer.AsyncAuditProducer - Send 20017K messages. Throughput: 123201.39 records/s
producer.AsyncAuditProducer - Percentile 0.99 : 2919.235584
producer.AsyncAuditProducer - Partition 0 : 3280836
producer.AsyncAuditProducer - Partition 1 : 8344435
producer.AsyncAuditProducer - Partition 2 : 8391752
```

I've tried with many different configurations, but it didn't help. The throughput/latency were on the same level as without 
availability timeout set. Decreasing it to below `40ms` started to negatively affect both latency and throughput. 
I've thought that maybe the higher percentiles will be better, but they were not. Here is the chart for 
p999 latency for 8 runs with adaptive availability timeout set to `50ms` vs without it.  

![adaptive-timeout-bench]({{site.baseurl}}/img/producer/adaptive-timeout-bench.png)

The throughput was always ~120K records/sec. The difference in messages sent to fast partitions vs the slowest was also 
always very similar to the one without `partitioner.availability.timeout.ms` set. Before 

### High level communication view



![kafka-tcp-protocol.png]()

Before any request can be sent, producer must establish a connection with the broker. But there are many brokers in the 
cluster, and connecting to each of them is not necessary until we have records to send to that broker. 


### KafkaProducer components
// TODO - explain role of each component + diagram

```
zwiększyć maksymalny rozmiar requesta na brokerze i sprawdzić throughput

### Tuning producer
1. We have to take into consideration a compression. Snappy/lz4 is better than gzip in terms of speed but compression ratio is worse.
2. Adding more user threads can help speedup compression because it is happening on the user's threads.
3. Each batch is sitting in a queue before grouping them by leader and sending to the broker.
4. Does the producer sends only ready batches? What about stealing (piggybacking) not ready batches?
5. What about max.in.flight.requests.per.connection? ordering? how throughput/latency is affected by that?
6. kafka-perf/ProducerPerformance tool
7. avg throughput = request_rate_avg * request_size_avg / compression_ratio
8. Theoretical upper request rate = (1000 / request_latency_avg) * number_of_brokers
9. Latency = record_queue_time_avg / 2(why?) + request_latency_avg + callback_latency
10. After running kafka-perf and getting throughput we can compare it to our network bandwidth to see if can improve.
10. throughput_avg = request_rate_avg * request_size_avg / compression_ratio, if we know theoretical 
11. Check if increasing request_size can help. Do it by:
    12. Add more threads -> so within a specific time window we can pack more data
    13. Increase number of partitions -> decreasing lock contention (how to check contention?)
    14. Increase linger.ms -> more batching (how linger.ms affects throughput and latency?) -> benchmark and chart needed
12. The results from one perf testing was that increasing `batch.size` caused a decrease in throughput because time to filling batch was inreasing.
    But increasing batch size can improve latency by preventing the batches from piling up.
13. The recipe is to finding a throughput bottleneck:
    14. if it is in users thread then increase user thread -> but it can cause contention
    15. if it is in sender thread and throughput is much lower than network bandwidth or record queue time is large or 
    the batch_size_avg is almost as batch.size
    16. if it is in broker that latency will be high
```
```
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
    6. During the switch KafkaProducer informs the partitioner to pick the new partition. May exists edge cases when the
       `linger.ms` passed, but the bytes sent < `batch.size`. In that case, we'll send the batch anyway, but don't immediately change
       partition. Max 2x `batch.size` can be sent in that case.
6. Sending messages
    5. By the way of calculating ready nodes to send data the KafkaProducer calculates here statistics for adaptive partitioning
       used in point 5.3.
        - checking if partitions have leaders
        - getting partitions queue size
        - if enabled adaptive partitioning with incorporating broker's latency, excluding slower brokers from sending data

       Obviously the real request is more complex, but this is the gist of it. Single producer instance can send multiple
       requests like that to multiple brokers at the same time.
    7. The response contains metadata about each batch for each topic partition. For each record in each batch KafkaProducer
       then call user provided callback with that metadata as an argument. Note that the callback is called in the same thread
       as the rest of producer's stuff, so don't use the blocking or long-running operations there.
```
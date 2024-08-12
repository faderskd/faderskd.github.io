---
layout: post
title:  "Kafka replication deep dive"
date:   2024-05-15
---

> [INFO]  
> The post covers replication for Kafka version (>=3.7).
{: .block-tip }

### Replication

Like any serious distributed storage, Kafka provides numerous ways of configuring durability and availability. 
You've heard it a million times, but I'll repeat it the million + 1 time: in a distributed system world some of the nodes will 
inevitably fail. Even if the hardware doesn’t break, there are still other scenarios: power outage, OS restart, network slowdown 
separating node from the rest of the system. Overcoming those issues in Kafka requires understanding different 
tunables. Every configuration involves awareness of the consequences. 
Our objective is to gain knowledge about the internals of Kafka replication, which will let us set up 
a system adjusted for expected requirements.  

### Topics, Partitions, Brokers

Kafka divides published messages into a set of topics. A topic is just an abstraction for one category of related messages. 
It's something like an SQL table or database collection. The topic is divided into partitions. Partitions are distributed 
across different machines (brokers). Partitions are a way to achieve scalability because clients (producers) can publish messages to 
many partitions in parallel. We'll cover it in a second. The messages are guaranteed to be read in the same order as 
they were written within a *single topic partition*.    

![kafka-topics.png]({{site.baseurl}}/img/replication/kafka-topics.png)

### How to choose number of partitions 

When choosing the number of partitions there is a general advice: calculate it. You can then set a bit higher number when 
anticipating growth. Keep in mind, that you can only increase the number of partitions. Kafka denies decreasing this number. 
The number of partitions most of the time depends on the speed of consumers consuming from Kafka. 
Before the calculation, let's assume the simplified statement: **that a single partition can be consumed only by one consumer**. 
Those of you familiar with consumer groups know that it is true only within a single consumer group but forget about it now. 
So for example, if a single consumer's max throughput is 100 m/s (messages per second) and expected incoming traffic to a 
Kafka topic is 500 m/s, you will need at least 5 consumers running in parallel. Because a single partition can be serviced by a single 
consumer, we'll need at least 5 Kafka partitions.

> [INFO]  
> I purposely focus only on the consuming side in the calculation. We didn't take into account if that number of partitions is enough
> for writing provided throughput. I assume that most of the time the bottleneck will be the consuming application rather than 
> Kafka itself. We can scale number of producers independently of the number of partitions, but the parallel consumers are limited by the 
> number of partitions. 
{: .block-tip }

The general advice is: `partitionsCount = expectedThroughput / throughputOfASingleConsumer`

### Increasing availability and durability - partition leaders and followers

Now we know that for better produce/consume scalability we can bump up the number of partitions, but what happens when one of the machines hosting 
a partition fails or restarts? You may think: "I can configure the `KafkaProducer` to omit the failed partition" from 
publishing. Well, maybe, but what about already published messages? What about consumers waiting for these messages? 
What if the disk crashed permanently? You feel we need something better - redundancy of course. 

Each partition is replicated to a configured number of replicas. Each partition has only one leader replica at a time. The leader 
can change depending on its availability. The producer publishes messages directly to the leader, and similarly, the consumers
fetch messages from the leaders. Although there is an option to consume from [followers](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica), 
we do not consider this option here. 

![replication-from-followers.png]({{site.baseurl}}/img/replication/replication-from-followers.png)

When creating a topic we provide at least two basic parameters: `partitions` and `replication factor`. The former is
a number of partitions a topic is divided into. The latter indicates the replicas count for each partition. 
The picture above has two topics with the following configuration:  
1. TopicA - partitions: 2 (P1/P2) replication factor: 2 (leader and one follower)
2. TopicB - partitions: 2 (P1/P2) replication factor: 2 (leader and one follower)

So for example, `Topic A/P1 (leader)` means a leader replica of partition P1 for topic A. 
As you may see, we have a few types of requests here. The `ProduceRequest` is used by the `KafkaProducer` to publish 
events to the partition leaders. The `FetchRequest` is used by both, the consumers, and follower replicas to consume events. 
Similarly to the consumers, follower replicas fetch messages from their partition leaders. 

A command to create the topic:
```postgresql
./kafka-topics.sh --create --topic A --bootstrap-server localhost:9092 --replication-factor 2 --partitions 2
```

A command to see topic details:
```postgresql
./kafka-topics.sh --describe --topic A --bootstrap-server localhost:9092

Topic: A        TopicId: D7DUpsdPTPqXnfigwjKHAg PartitionCount: 2       ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: A        Partition: 0    Leader: 2       Replicas: 2,3   Isr: 2,3
        Topic: A        Partition: 1    Leader: 3       Replicas: 3,1   Isr: 3,1
```

For now, just look at the `Topic: A        Partition: 0    Leader: 2       Replicas: 2,3` part of the output. 
That means that partition 0 of topic A has two replicas situated on broker machines with IDs 2 and 3, 
and that the leader replica is a broker 2. I'll describe the meaning of `Isr` later. 

### Kafka brokers and controller

So far, we know that Kafka divides its topics into multiple partitions and that each partition is replicated to the 
configured number of replicas. We know that partitions are distributed across the cluster's brokers. We don't know yet 
how they are distributed and when the partition to broker assignment happens. Kafka has two types of brokers: a regular one and 
a controller. The regular broker hosts partition replicas, serving client's requests (like produce/fetch requests). The 
controller, on the other hand, manages the cluster. At this moment, it is important to know that: **every time you execute the 
topic creation command, the request is directed to the controller and the controller assigns partitions of a topic to the 
specific brokers. The controller chooses a leader for partition too**. 

So how does the controller assign partitions to the brokers? The actual algorithm takes into consideration both, the number of 
brokers and the racks where they are situated in a server room. Assume that we 
have all brokers on the same rack. Kafka controller tries to distribute partitions evenly, so each partition is on 
a different broker. That's why it spreads partitions using a round-robin algorithm, and for each next partition it offsets the starting broker, 
so each leader is on a different broker. If you're curious about the details you can read it [here](https://github.com/apache/kafka/blob/5552f5c26df4eb07b2d6ee218e4a29e4ca790d5c/metadata/src/main/java/org/apache/kafka/metadata/placement/StripedReplicaPlacer.java#L72).

Let's say we want to create a `test` topic with 3 partitions, each replicated across 3 replicas. 
We have also 3 brokers with ids: 1, 2, 3.

```postgresql
             ----------------- The first node in assignment is a leader.
             |         
partition 0: 1, 2, 3
partition 1: 2, 3, 1
partition 2: 3, 1, 2
```

The first node in an assignment is a leader. The placement assigns brokers to partitions
using a round-robin. Leaders offset by one for each new partition to avoid choosing
the same broker for each partition's leader.

How does it look like in reality?
```postgresql
./kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --replication-factor 3 --partitions 3
Created topic test.

./kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092

Topic: test     TopicId: ha0jRlepRvasAGLxmyx60A PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3       Isr: 1,2,3
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,3,1       Isr: 2,3,1
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2       Isr: 3,1,2
```

### Cluster Metadata

We've discovered the topic structure and how the partitions are assigned to the different brokers. The next puzzle 
is how the clients know where to send and fetch messages from. As we've mentioned, `ProduceRequest` and `FetchRequests` 
go to the partitions leaders. To find them, clients send `MetadataRequest` to any broker in a cluster. Each broker 
keeps cluster-wide metadata information on a disk. The usual brokers learn about metadata changes from... Kafka controller. 
Every admin change e.g. increasing the number of partitions for a topic goes to the controller. The controller 
propagates the change to the usual brokers via metadata fetch API, and then clients fetch that metadata from the brokers.

![metadata-protocol.png]({{site.baseurl}}/img/replication/metadata-protocol.png)

In the picture above, the brokers use `FetchRequests` to get updates about metadata. Internally, metadata is a topic and 
brokers use the existing API between them to fetch the newest metadata information. Producers and consumers, on the other
hand, use dedicated `MetadataRequest` for receiving metadata information. They request information about interested
topics, their partitions, and leaders. Details can be found [here](https://kafka.apache.org/protocol.html#The_Messages_Metadata)
(scroll to the latest version). 

### Replication from leader

As I mentioned, the follower replicas use `FetchRequest` to request messages from the leader. For now, 
assume that the only parameter is offset. So `FetchRequest(offset=x)` means that:
1. the requesting replica wants messages with offsets `>= x`
2. the requesting replica confirms that it persisted locally all messages with offsets `< x` (and now it wants more)
3. the leader replica returns messages with offset `>= x` to the follower, and saves in its internal state that the
requesting follower has all messages with offsets `< x`.

Each of the brokers replicates messages independently. Each of them may have a different set of messages replicated at a 
specific point in time. 
The end of the broker's log for a particular partition is called `LEO` (Log End Offset). It indicates that the broker/replica has
messages with offsets up to the  
`LEO - 1`. So `LEO` is the first offset of the partition that the replica doesn't have yet. For example, if the 
replica has a log of messages with offsets `[0,1,2,3,4,5]`, its `LEO=6`.

### Replicas vs In-Sync Replicas
As I explained previously, once the `KafkaProducer` sends the `ProduceRequest` to the partition leader, the broker saves 
the message, and then follower replicates by fetching it using `FetchRequest` API. Let's take as an example the 
previously created topic with 3 partitions. For partition `0` there are 3 replicas `Replicas: 1,2,3`. The leader is `1`, the
followers are `2` and `3`. Each of the followers has a different pace of replication as they are independent nodes. While replicating, 
the leader tracks each of the follower's replication progress. It can assess which of them are fast and which can't keep up. 
To recall, the topic partition is a log of records, and each record has its offset.

![partition-with-offsets.png]({{site.baseurl}}/img/replication/partition-with-offsets.png)

The leader tracks each of the follower's fetching offset.  


![leader-tracking-followers.png]({{site.baseurl}}/img/replication/leader-tracking-followers.png)

We can now comprehend the concept that Kafka heavily depends on to ensure the safety and correctness of replication: `in-sync-replicas` (ISR).
For each topic partition, Kafka maintains a curated set of nodes, which are in sync. At a high level, these are the **healthy** brokers 
that **can keep up with fetching the newest messages** from the leader. The [documentation](https://kafka.apache.org/documentation/#replication)
says that to be in sync:

```postgresql
1. Brokers must maintain an active session with the controller in order to receive regular metadata updates.
2. Brokers acting as followers must replicate the writes from the leader and not fall "too far" behind.
```

And there are two broker configs for these conditions:
1. [broker.session.timeout.ms](https://kafka.apache.org/documentation/#brokerconfigs_broker.session.timeout.ms) -
   The regular brokers inform the controller that they are alive via heartbeat requests. If
   the time configured as this config passed between two consecutive heartbeat requests, the broker is not in sync
   anymore.
2. [replica.lag.time.max.ms](https://kafka.apache.org/documentation/#brokerconfigs_replica.lag.time.max.ms) -
   Each time the same broker fetches messages from a specific topic partition's leader, and it gets to the end of the log,
   the leader records the timestamp when that happened. If the time that passed between the **current time** and the **last fetch 
   that got to the end of the log** is greater than `replica.lag.time.max.ms`, the replica hosted by the broker is thrown out 
   of in-sync replicas set. 

![leader-isr-state.png]({{site.baseurl}}/img/replication/leader-isr-state.png)

The second timeout is a little bit harder to understand. The picture above shows that replicas R2 and R3 fetch from different 
positions from the leader's log. The `Partition state` is a global view of the partition in a Kafka cluster. 
The `Leader state` is calculated locally by the partition leader. Each time the replica got to the end of the log 
(`FetchRequest{offset=100}` in this case) the leader sets the `lastCaughtUpTimeMs` timestamp for that replica. 
Then, the leader periodically checks if each replica fulfills ISR condition:
`now() - lastCaughtUpTimeMs <= replica.lag.time.max.ms`. If not, that replica is thrown out of the ISR. For now, just assume 
that the leader somehow changes the global Kafka view of that partition. So for example, if replica `R2` falls out of ISR, the 
global view of the partition would be like: `Replicas: [R1, R2, R2], Leader: R1, ISR = [R1, R3]`.

### Consumer's message visibility, watermark and committed messages.

The question that comes to mind when thinking about replication is: when the consumers can see the published message?
The answer: **when all in-sync replicas replicate the published message**. Let's investigate what that exactly means.

![committed-messages.png]({{site.baseurl}}/img/replication/committed-messages.png)

The picture above contains a topic with a replication factor 3. The producer published 5 messages with offsets: `0,1,2,3,4`. 
The messages `0,1,2` were already replicated by all current in-sync replicas. The messages `3,4` were replicated only by one 
follower, so 2/3 of in-sync replicas have the copy. The offset below which all messages were replicated by all in-sync 
replicas is called a **high watermark**. Everything below that threshold 
is visible to the consumers. So if the high watermark is at offset `X`, all messages visible to the consumers are `< X`. 
That part of the log is also called the **committed log**. Consider the high watermark as an indicator of the safely 
stored part of the log. This is an important thing to understand how Kafka provides high reliability in different failure 
scenarios. We'll get to that.

Let's find out how and when exactly the high watermark is moved forward. 

We have the following setup:

```postgresql
TopicA - partitions: 1 (one partition P1 for simplicity), replication factor: 3 (leader and 2 followers)
leader = R1
replicas = [R1, R2, R3]
ISR (in-sync replicas) = [R1, R2, R3]
```

I'll use the marker `T[X]` for designating the time progression. Remember that `LEO` is the log end offset, and it is the next offset 
that the replica doesn't have yet for partition `P`. The leader moves the high watermark, and it tracks each
of the follower's `LEO`. I purposely show `LEO` and watermark only for a leader. In reality, each of the replicas track 
its own `LEO` and copy high watermark from the leader. I will omit it for simplicity though.

```postgresql
----------------------------------T1-------------------------------------------------
KafkaProducer published 3 messages with offsets 0,1,2 to the leader R1.

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = []
R3          | log = []
----------------------------------T2-------------------------------------------------
Replica R2 sends FetchRequest(offset=0) to the leader, and the leader responds with 
messages it has in log. Note that R2 LEO doesn't change as the leader hasn't yet 
been confirmed about persisting the messages by the follower. It has to wait for 
the next FetchRequest(offset=3) to be sure that previous offsets were saved. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = []
----------------------------------T3-------------------------------------------------
The replica R3 sends FetchRequest(offset=0) to the leader, the leader responds 
with messages it has in the log. The same situation as above. The replica didn't confirm 
getting the messages yet. The high watermark cannot move forward. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T4-------------------------------------------------
KafkaConsumer (client app) sends FetchRequest(offset=0) to the leader. The leader 
doesn't respond with any messages because the high watermark is 0.
----------------------------------T5-------------------------------------------------
The replica R2 sends FetchRequest(offset=3) to the leader, the leader has no messages 
having offset >= 3, so it doesn't return any messages. But now the leader knows that 
replica R2 persisted messages with offset < 3. The R2 LEO changes, and points to the 
next message offset not present in the R2 log. The high watermark cannot move forward 
since there is still one in-sync replica R3, which didn't confirm getting the message. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 3, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T6-------------------------------------------------
The replica R3 sends FetchRequest(offset=3) to the leader, the leader has no messages 
having offset >= 3, so it doesn't return any messages. But now, the leader knows that 
replica R3 persisted messages with offset < 3. The R3 LEO changes to 3. All in-sync  
replicas reached to the offset 3 (have their LEO=3). The high watermark can move forward 
because all in-sync replicas confirmed getting messages with offset < 3. To be more 
precise: the high watermark can progress when all in-sync replicas' LEO exceeds the 
current high watermark. Because the HighWatermark = 0 and R1 LEO = 3, R2 LEO = 3, 
R3 LEO = 3, the leader can set the watermark to 3 as well. The [0,1,2] is now a committed
log and is visible to the KafkaConsumer clients. 

R1 (leader) | log = [0,1,2], HighWatermark = 3, R1 LEO = 3, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]

----------------------------------T7-------------------------------------------------
KafkaProducer published 2 more messages with offsets 3,4 to the leader R1.

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 5, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T8-------------------------------------------------
The replica R2 sends FetchRequest(offset=3) to the leader, the leader responds 
with 2 messages [3,4] having offset >= 3. R2 LEO does not change on the leader 
because it lacks the message persistence confirmation - the leader has to wait 
for FetchRequest(offset=5) from R2 to confirm saving [3,4] messages. The high watermark 
doesn't move forward. It can only progress when all in-sync replicas' LEO exceeds 
the current watermark. 

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 5, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2,3,4]
R3          | log = [0,1,2]
----------------------------------T9-------------------------------------------------
The replica R2 sends FetchRequest(offset=5) to the leader, the leader hasn't any 
messages with offset >= 5. The leader changes its state for R2 LEO to 5 as it got 
confirmation for saving messages with offset < 5. The high watermark does not 
change as there is still a replica R3, which is in sync, and its 
LEO <= current HighWatermark. 

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 5, R2 LEO = 5, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T10-------------------------------------------------
KafkaConsumer (client app) sends FetchRequest(offset=0) to the leader. The leader 
responds with messages having offset < HighWatermark: [0,1,2] which is a committed log.
```

### Shrinking ISR

We said that in order to advance the watermark, all in-sync replicas must replicate the message. But what happens if one of 
the replicas becomes slow or even unavailable? Effectively, it can't keep up with a replication. Without special treatment, 
we would end up with an unavailable partition - the HW couldn't progress, and consumers couldn't see the new messages. We 
can't just wait for a replica until its recovery, this is not something a highly-available system would do. 
Kafka handles such a situation by shrinking the ISR. Do you remember the two [conditions](#replicas-vs-in-sync-replicas) 
for a replica to stay in sync? I promised to explain what exactly happens when the replica falls out of the ISR. 
There are basically two scenarios when that happens:  

1) The unhealthy broker stops sending heartbeats to the controller. The controller fences such a broker. That means it 
is removed from the ISR of all its topic partitions. This implies losing leadership for these partitions too. 
If the fenced broker is a leader for any partitions it handles, the controller performs leader election, and picks one of 
the brokers from the new ISR as a new leader. That information is then disseminated across the cluster using metadata protocol 
(from controller -> to other brokers -> and to clients). 

![failed-leader-heartbeat.png]({{site.baseurl}}/img/replication/failed-leader-heartbeat.png)

The picture above shows the point in time when the current leader's broker stops sending heartbeats [1]. The controller detects it
and calculates the new cluster metadata [2].

![controller-performs-leader-elections.png]({{site.baseurl}}/img/replication/controller-performs-leader-elections.png)

The metadata then goes to the followers, so they can start fetching from the new leader. For picture clarity, I 
purposely didn't draw metadata fetching requests from brokers to the controller and from consumers to the brokers. 

2) The second party allowed to modify the partition's ISR (besides the controller) is a partition leader. 
The leader removes slow or unavailable replicas from the ISR. It then sends the `AlterPartitionRequest` to the controller 
with the new proposed ISR. The controller saves/commits that information in the metadata, which is then spread across the cluster. 

![follower-cant-keep-up-to-leader.png]({{site.baseurl}}/img/replication/follower-cant-keep-up-to-leader.png)

Once the follower (Broker2) falls too far behind the leader [1], the leader removes it from the partition's ISR [2], 
and sends the `AlterPartitionRequest` to the controller [3].

![leader-commits-metadata-throwing-follower-out-of-isr.png]({{site.baseurl}}/img/replication/leader-commits-metadata-throwing-follower-out-of-isr.png)

Then metadata gets around the rest of the brokers and clients.

### Expanding ISR

Assuming the partition currently has a leader, adding a replica to the ISR can be only done by that leader. Once the 
out-of-sync replica catches up to the end of the leader's log, it is added to the ISR by the leader issuing the `AlterPartitionRequest` 
to the controller. 
What if there is no leader for partition? It can be the case that all replicas for example restarted. In this 
case, the controller elects a leader and then saves that information in metadata (that will eventually go to the replicas). 
The rest of the replicas will join the ISR by catching up to the new leader. 


###  Min-ISR (minimum in-sync-replicas) and AKCs (acknowledgements)

In a distributed systems world, sooner or later you will come across a choice between 
availability or consistency during failure scenarios. In Kafka, this choice is made by a collaboration of the 
client - `KafkaProducer`, and the Kafka broker itself. Until now, we've mentioned that the `KafkaProducer` just publishes 
messages to the leader, and then the replicas pull the messages, so they eventually end up on a few independent brokers. 
We haven't specified yet when the producer should state that the message was written to the topic:  
1. When just the leader saves the message on a local machine without waiting for replication from the followers? What if the 
leader crashed a short time after acknowledging, but before the replication happened?
2. When all ISR replicas replicate the message? We know, that the ISR is a curated list of currently available and healthy partition replicas. 
We know it is dynamic. It can be even limited to the leader alone. It can be even empty!  
3. When some minimal number of ISR confirmed? How many brokers should get the copy, so we consider it safely replicated? This 
is what a Kafka topic `min.in.sync.replicas` property configures. The documentation can be found [here](https://kafka.apache.org/documentation/#topicconfigs_min.insync.replicas).

The Min-ISR tells the partition's leader, for how many of the ISR (including itself) it should wait before 
giving a positive acknowledge response to the client. But this is not the end, I said that the `KafkaProducer` 
participates in the decision-making process. That's why we have the following producer configuration: [acks](https://kafka.apache.org/documentation/#producerconfigs_acks).
I recommend reading the documentation, but in case you don't:
1. `acks=0` - it is a fire-and-forget type of sending. We don't care about saving the message in Kafka. We assume that most 
of the time the cluster is working fine, and most of the messages will survive. 
2. `acks=1` - wait until the replica leader stores the message in its local log, and don't care if it was replicated 
by followers.
3. `acks=all` - wait until the message is written to the leader and all in-sync replicas. If the number of ISR 
falls below Min-ISR the producer gets an error. 

Note that in any case of `acks=0/1/all`, the consumers see messages only after they are committed. This means 
that the leader waits for all current ISR to advance the high watermark, and we know that consumers see messages 
only up to the HW. Additionally, if the condition `Min-ISR >= ISR` is not met, the watermark cannot advance. 
So even if message was published with `ack=0/1`, it is available for consumers when all ISR replicated up to that 
message offset and the number of ISR is at least Min-ISR. 

### How Kafka persists messages

By default, Kafka broker doesn't immediately flush received data directly to the disk storage. It stores the
messages in the OS I/O memory - [page cache](https://en.wikipedia.org/wiki/Page_cache). 
This can be controlled via `flush.messages` setting. 
When set to `1` Kafka will flush every message batch to the disk and only then acknowledge it to the client. But then 
you can see a significant [drop in performance](https://www.confluent.io/blog/kafka-fastest-messaging-system/#fsync). 
The recommendation is to leave the default config - thus not doing `fsync`. That means the message confirmed by the leader (`acks=1`) is not 
necessarily written to disk. The same applies for `acks=all` - all `ISR` replicas confirmed by only storing the message in the local OS page cache. 

![page-cache.png]({{site.baseurl}}/img/replication/page-cache.png)

The OS eventually flushes the page cache when it becomes full. The flushing also happens during the graceful OS shutdown. 
Because of this asynchronous writing nature, the acknowledged, but not yet flushed messages can be lost when the broker 
experiences a sudden failure. Note, that page cache is retained when the Kafka process fails. The page cache is managed 
by the OS. The abrupt entire OS shutdown is something which can cause the data loss. 
Leveraging the page cache provides improved response times and nice performance. But is it safe? And why I'm talking about 
it during the replication? Kafka can afford asynchronous writing thanks to the usage of its replication protocol. 
Part of it is partition recovery. In the event of an uncontrolled partition's leader shutdown, where some of the messages may 
have been lost (because done without `fsync`) Kafka depends on the controller broker, which performs leader election for that partition. 
The objective is to choose a leader with a complete log - the one from the current `ISR`. 

### Preferred leader and leader failover

Each Kafka topic's partition has a leader replica initially chosen by the controller during a topic creation. 
I previously explained the algorithm for assigning the leaders to brokers. The leader assigned at the beginning is called 
a preferred leader. It can be recognized by examining the output from Kafka topic details.

```postgresql
 ./kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic=test

Topic: test     TopicId: 2ZF_sUf2QjGHA5UJDFbY1g PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test     Partition: 0    Leader: 2       Replicas: 2,3,4 Isr: 2,3,4
        Topic: test     Partition: 1    Leader: 3       Replicas: 3,4,1 Isr: 3,4,1
        Topic: test     Partition: 2    Leader: 4       Replicas: 4,1,2 Isr: 4,1,2
                                                                  ^
                                                                  |
                                              preferred leader ----
```

The preferred leader is the one placed at the first position in the `Replicas` list. The leadership can change. These changes are triggered 
by a few actions, and one of them is leader shutdown. During the graceful/controlled shutdown, the broker de-registers itself 
from the controller by setting a special flag in the heartbeat request. Then the controller removes the broker from any partition's `ISR` 
and elects new leaders for partitions previously led by this broker. In the end, the controller commits this information to the metadata log. 
The new leaders will eventually know about the changes via 
the metadata [propagation](#cluster-metadata) mechanism. In the same way, followers find out from which node to 
the replicate partition's data. 
And the same happens for clients too - they use metadata to discover new leaders. 

![clean-broker-shutdown.png]({{site.baseurl}}/img/replication/clean-broker-shutdown.png)

1. Followers (`Broker2`/`Broker3`) fetch from `Broker1` being an old leader.  
2. `Broker1` sends periodic `HeartbeatRequest` to the controller. Once it wants to shut down gracefully, 
it sets a special `wantShutdown` flag.  
3. The controller seeing this flag iterates over all partitions with `ISR` containing `Broker1`. Then it removes the broker from 
the `ISR`, and if it was also a leader, elects a new one. Then it writes these changes to the metadata log.  
4. `Broker2`/`Broker3` learns about the changes by observing the metadata log via periodic fetch requests from the controller.  
5. Once the `Broker3` detects a new leader, it starts pulling from it.
6. Now `Broker1` can shut down.  

Another reason for changing a leader is the current leader failure. The failed broker stops sending 
heartbeat requests to the controller. Once the `broker.session.timeout.ms` passed, the controller removes the leader from 
any partition's `ISR`, elects a new leader from the rest of the `ISR` and commits that change to the metadata log. 

### Unclean leader election

Consider a topic with a `replication.factor=4`, `min.in.sync.replicas=2` and single partition `A` for simplicity. 
The current partition `ISR=2`. We have two brokers (Broker2/Broker3) with performance degradation, 
so they end up outside the `ISR`. The `ISR >= MinISR` condition is met, so any `ACK=1/all` produce requests work fine. 

![performance-degradation-1.png]({{site.baseurl}}/img/replication/performance-degradation-1.png)

What happens when the 3rd broker (Broker4) crashes?  
1. The producers with `acks=all` get errors because `ISR < MinISR`.  
2. The `acks=1` still works fine.  
3. The high watermark cannot progress because `ISR < MinISR`. No more messages can be [committed](http://127.0.0.1:4000/2024-05-15-kafka-replication.html#consumers-message-visibility-watermark-and-committed-messages). 
Because of that, consumers don't get any new messages. 
4. Effectively the partition is available only for writes with `acks=1`, but not for reads.  

![performance-degradation-2.png]({{site.baseurl}}/img/replication/performance-degradation-2.png)

What happens now when the current leader (Broker1) fails? It was the last in sync replica. The leader election described so far 
assumed that the next leader can be only selected from the `ISR` set (clean election) - but there are no more `ISRs`. 
The partition becomes completely unavailable. What choices do we have in this situation?

![performance-degradation-3.png]({{site.baseurl}}/img/replication/performance-degradation-3.png)

1. We care about not losing committed data, and we have to wait for at least one of the previous `ISR` replicas to become 
available again. Which are these? The last leader and the broker which failed before (Broker1/Broker4) 
because then the `ISR >= MinISR`. This is a broad topic described as the [Eligible Leader Replica](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966%3A+Eligible+Leader+Replicas), 
and it is currently under development.  
2. We care about making the partition available even if it means losing some data. We still have 2 replicas (Broker2/Broker3) which 
were outside the `ISR` from the outset and may not have all committed events. But at least they are alive, so we can 
elect one of them as the new leader. We can enable this by setting [`unclean.leader.election.enable=true`](https://kafka.apache.org/documentation/#topicconfigs_unclean.leader.election.enable) 
on a topic.  

### Availability vs consistency

Setting all previously mentioned configurations:

* number of acknowledgements `acks`
* minimum in-sync-replicas (`min.in.sync.replicas`)
* number of replicas (`replication.factor`) 
* unclean leader election (`unclean.leader.election.enable`)

requires understanding how they affect availability and consistency. Before configuring anything, start with answering the 
question: do you accept losing any data ? Do you accept that the produced message may never be delivered to the consumers 
even when Kafka responded successfully during publishing ? Most of the time, Kafka will work fine, but you have to prepare 
for the hard times too:
* broker disk failures
* slow network
* network partitions (some of the brokers cannot communicate with others)
* broker ungraceful shutdown
* broker machine disaster

What is more important to you when the above conditions arise ?

**Availability** - being able to produce messages despite the fact that some of them may be permanently gone  
**Consistency** - stop producing and throwing errors when there is not enough replicas to provide expected durability

When you know the answer, then you can lean toward set of configuration adjusted to these requirements.  

**Availability:**
1. **`acks=1`** - only leader confirms during producing, no need to wait for higher number of replicas before acknowledging 
the messages. Note that this concerns only write availability. The leader does not return any messages to the consumer until 
they are replicated to the current `ISR`. If the `ISR < Min-ISR` then the high watermark cannot progress, and the consumption 
stops despite successful writing.
2. **lower `min.in.sync.replicas`** - as mentioned in the previous point, for `acks=1`, the `Min-ISR` does not affect producing, but will 
always affect consuming. The consumers can only fetch committed messages. These are messages with offsets below the high watermark,
replicated to at least `Min-ISR` number of brokers (the high watermark progression depends on condition `ISR >= Min-ISR`). 
> Before blindly just setting it to 1, note that while higher `Min-ISR` puts a restriction (and delay) on a consuming process, 
> it also increases predictability for consumers. They get
> messages only once they are stored on multiple machines. For example, reprocessing the same topic multiple times, interleaved with the
> brokers failure in the meantime, results in a more consistent view of messages.
If you want to use `acks=all`, lowering `min.in.sync.replicas` will increase the producing availability as well. 
3. **higher `replication.factor`** - the more replicas you have, the more of them can serve as the `ISR`. For `ack=1`, higher 
number of `ISR` means broader choice of the valid replica during leader failover. For `acks=all`, more `ISR` means greater chances 
that `ISR >= MIN-ISR` while some of the replicas are unavailable. However, when all nodes are healthy (thus are in `ISR`), 
more replicas means waiting for a higher number of `ISR` replicas before committing the message. And obviously, brokers has 
more work to do because of replication. 
4. **unclean.leader.election.enable** set to `true` - when the subset of nodes with a complete committed data (`ISR`) failed, you 
favor electing not up-to-date (outside of `ISR`), but alive node as the leader, rather than waiting for one of the `ISR` to recover. 

**Consistency**
1. **`acks=all`** - leader waits for all replicas in the current `ISR` before acknowledging to the client.  
If `ISR < MinISR` then it
returns errors. The visibility for consumers and high watermark progression is the same as for `acks=1`
2. **higher `min.in.sync.replicas`** - should be lower than `replication.factor` to have some space for availability as well 
3. **higher `replication.factor`** - same as in the availability section 
4. **unclean.leader.election.enable** set to `false` - only replicas with a complete committed log (`ISR`) can become a leader. 
When no currently available, the whole partition is nonfunctional, but will not lose data (when at least one of the `ISR` recovers).

### Performance 

We graded different configurations in terms of availability and consistency, but they have impact on the performance as well:  
1. **`acks=1`** - causes better write latency as the leader doesn't wait for all `ISR` to confirm messages. The consumer 
still has to wait for full `ISR`.
2. **`acks=all`** - increased write latency. On consumers side nothing changes.
3. **`replication.factor`** - higher number of replicas indicates higher network, storage and CPU utilization. The brokers 
have much more work to do because of the replication. And finally, the end-to-end latency will increase too.
4. **`min.in.sync.replicas`** - this does not affect the performance, the leader waits for all `ISR` when producing 
with `acks=all` and when moving high watermark in any case. The `MinISR` does not affect write, end-to-end latency or 
throughput. 

// TODO maybe picture from here: https://www.confluent.io/blog/configure-kafka-to-minimize-latency/.

### Configurations examples

I'd like to provide you a few examples showing the implications of mixing different configurations. 
The `unclean.leader.election.enable` is `false`. 

1. **`replication.factor=4`, `min.in.sync.replicas=3`, `acks=all`** - this config provides high consistency at the cost of 
availability and write latency. Each message is replicated to **at least** 3 brokers (including leader) before getting success response. 
During the healthy broker's conditions, the `ISR=4`, so `ack=all` will wait for all 4 brokers. One of the brokers can 
fail (`ISR=3`), and we still have `ISR >= Min-ISR` condition met, so we still will be able to produce new messages. 
Losing one more broker (`ISR=2`) affects availability, as the `ISR >= Min-ISR` won't apply anymore.  
When the `ISR=3`, and the partition is still available 
for writes, it can survive up to 2 further replicas failures without losing data. In such an extreme situation, there still would be one node with a complete *committed* log, which would be elected as 
the leader, so rest of the brokers could replicate from it after the recovery.  
In a more common scenario, (`ISR=4`) we would survive up to 3 replicas crashes.  
To sum up: we can tolerate up to 1 broker crashes while still being available for writes, 
and then 2 (`Min-ISR - 1`) failures without losing data.  
2. **`replication.factor=4`, `min.in.sync.replicas=2`, `acks=all`** - moderate consistency and availability. We can tolerate 
up to 2 unhealthy replicas while providing availability, but then we can lose only one additional partition (`Min-ISR - 1`) 
without losing data.  
3. **`replication.factor=4`, `min.in.sync.replicas=2`, `acks=leader`** - we favour availability over consistency. The leader 
responds successfully after storing messages in its local log, before replication to the rest of the `ISR` happens. 
By using this configuration, you accept the possibility of losing some events. The lost are those acknowledged by the leader,
but still not replicated by the `ISR` - which is a non-committed part of the log. The reality is that you can lose messages in more scenarios 
than just leader failure. As the author of the [post](https://jack-vanlightly.com/blog/2018/9/14/how-to-lose-messages-on-a-kafka-cluster-part1) 
shows, the networking problems are even worse (the post investigates Kafka with Zookeeper). Nevertheless, once we accept the possibility of occasional data loss, 
we get very high availability: we can run even without 3 brokers. Note that consumers will see data once it is 
replicated by at least `Min-ISR=2` brokers.  
4. **`replication.factor=4`, `min.in.sync.replicas=1`, `acks=all`** - we favor availability over consistency in case of emergency,
but when the cluster is healthy, we want to replicate to a higher number of replicas - current `ISR`. 
When all replicas perform well, the `ISR=4`, and leader will acknowledge events when all `ISR` replicate.
The messages are then available for consumers after replication to the full `ISR=4`. In case of emergency, the `ISR` can even 
shrink to the leader itself, and we still be able to produce/consume (because of `Min-ISR=1`).
This may seem like a non-practical scenario: why to enforce higher consistency when everything goes well, but when not, 
just forget about safety and accept losing data ? Compared to any `acks=leader` configuration, this will be more reliable during 
leader restarts.  
For example, consider a scenario when the leader rebooted ungracefully, and the rest of the replicas was fine at that time. The leader 
could acknowledge some messages before the restart, by writing them to the page cache, which is not immediately synchronized to disk. 
If the producing was made with `acks=leader`, the leader didn't wait for `ISR` before acknowledging, which when followed by the 
shutdown could end up with a data loss. But in our case, the producer used `acks=all`. The leader waited for all current `ISR` 
before acknowledging. This means that the new leader from the current `ISR` had to have all acknowledged messages - no data loss.

### Conclusions

While we've covered a lot of replication stuff, it is still tip of the iceberg. There are plenty of mechanisms that Kafka 
implements to provide safety, availability and good performance. My focus was on providing necessary information for making 
conscious decisions while configuring Kafka topics and clients. In case you are interested in discovering the lower parts 
of this iceberg, I recommend to read the following materials:
1. Kafka replication [documentation](https://kafka.apache.org/documentation/#replication).   
2. [Kafka improvement proposal](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966%3A+Eligible+Leader+Replicas) 
(actually in progress) enhancing replication correctness.  
3. Really interesting (and detailed) [set of posts](https://github.com/Vanlightly/kafka-tlaplus/tree/main/kafka_data_replication/kraft/kip-966/description) about internals of Kafka replication.  
4. [Series of blog posts](https://jack-vanlightly.com/blog/2018/9/14/how-to-lose-messages-on-a-kafka-cluster-part1) about scenarios when Kafka 
can lose messages (these are for Kafka version before introducing KRaft controller).  
5. A [talk](https://www.confluent.io/en-gb/kafka-summit-sf18/hardening-kafka-replication/) describing how hard it was to implement safe Kafka replication protocol. The author says about different mechanisms 
preventing from losing messages.  
6. Replication for Kafka control plane - [KRaft](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum). 
We talked about replication between usual brokers, but avoided discussion about replication between Kafka controller nodes.  
7. [Why Kafka does not need fsync to be safe](https://jack-vanlightly.com/blog/2023/4/24/why-apache-kafka-doesnt-need-fsync-to-be-safe).  
8. And of course Apache Kafka [Github repository](https://github.com/apache/kafka/tree/trunk). 
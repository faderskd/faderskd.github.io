---
layout: post
title:  "Kafka replication deep dive"
date:   2024-05-15
---

> [INFO]  
> The post covers the replication for Kafka version using KRaft (>=3.0).
{: .block-tip }

### Replication

As any serious distributed storage, Kafka provides a numerous ways for configuration of a durability and availability. 
You've heard it a million times, but I'll repeat it the (million + 1) time: in a distributed system any node will 
inevitably fail. Even if the hardware won't break, the power outage, OS restart, network slowdown can take a node 
down or partition it from the rest of the system. Overcoming those issues in Kafka requires understanding different 
tunables. Every configuration brings a tradeoff, every tradeoff requires awareness of the consequences. 
Hence, our objective is to gain the essential internals of Kafka replication. This knowledge will enable us to set up 
the system in a more informed and confident manner. 

### Topics, Partitions, Brokers

Kafka divides a messages published to a set of topics. A topic is just an abstraction for one category of related messages. 
It's something like SQL table or database collection. The topic is divided into partitions. Partitions are distributed 
across different machines (brokers). Partitions are a way to achieve scalability because the clients (producers) can publish messages to 
any of the partition in parallel. We'll cover it in a second. The messages are guaranteed to be read in the same order as 
they were written within a *single topic partition*. I'll write a different post about details of producing and consuming from Kafka.  

// TODO: obrazek z producerem publikującym na partycje i konsumującym (P2)

### How many partitions 

When choosing the number of partition there is a general advice: calculate it. You can set then a little higher number when 
anticipating growth. Remember that you can only increase the number of partitions. Kafka denies decreasing this number. 
The number of partitions will depend on the speed of consumers consuming from Kafka. Before we do the calculation let's come 
up with a simplified statement: **that a single partition can be consumed only by one consumer**. Those of you 
who are familiar with consumer groups, will now that it is true only within a single consumer group, but forget about it now. 
So for example if a single consumer's max throughput is 100 m/s (messages per second) and expected incoming traffic to a 
Kafka topic is 500 m/s, you will need at least 5 consumers running in parallel. Because single partition can be serviced by a single 
consumer we'll need at least 5 Kafka partitions.

### Increasing availability and durability - Partition leaders and followers

Now we know that for better scalability we can bump up number of partitions, but what happens when one of the machines hosting 
a partition fails or restarts ? We can say, "ok I can configure the `KafkaProducer` to omit the failed partition" from 
publishing. Well, maybe, but what about messages already published ? What about consumers waiting for these messages ? 
What if the disk crashed permanently ? You feel we need something better - redundancy of course. 

Each partition is replicated to a configured number of replicas. Each partition has only one leader at a time. The leader 
can change depending on its availability. The producer publishes messages directly to the leader, and similarly, the consumers
fetch messages from the leaders. Actually there is an option to consume from [followers](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica), 
but we do not consider this option here. 

// TODO obrazek (P2)

When creating a topic we provide at least to basic arguments: `partitions` and `replication factor`. The former is
a number of partitions a topic is divided to. The latter is a number of replicas for each partition. 
The picture above has two topics with the following configuration:  
1. TopicA - partitions: 2 (P1/P2) replication factor: 2 (leader and one follower)
2. TopicB - partitions: 2 (P1/P2) replication factor: 2 (leader and one follower)

So for example a `Topic A/P1 (leader)` means a leader replica of a partition P1 for a topic A. 
As you may see we have a few types of requests here. The `ProduceRequest` is a one used by a `KafkaProducer` to publish 
events to the partition leaders. The `FetchRequest` is used both by the consumers and follower replicas to consume events. 
Similarly to the consumers, follower replicas fetch messages from its partition leaders. 

A command to create a topic:
```postgresql
./kafka-topics.sh --create --topic A --bootstrap-server localhost:9092 --replication-factor 2 --partitions 2
```

A command to see the topic details:
```postgresql
./kafka-topics.sh --describe --topic A --bootstrap-server localhost:9092

Topic: A        TopicId: D7DUpsdPTPqXnfigwjKHAg PartitionCount: 2       ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: A        Partition: 0    Leader: 2       Replicas: 2,3   Isr: 2,3
        Topic: A        Partition: 1    Leader: 3       Replicas: 3,1   Isr: 3,1
```

For now just look at the `Topic: A        Partition: 0    Leader: 2       Replicas: 2,3` part of the output. 
That means that partition 0 of a topic A has two replicas situated on broker machines with ids 2 and 3, 
and the leader replica is a broker 2. I'll describe the `Isr` meaning later. 

### Kafka brokers and controller

So far we know that Kafka divides its topic into multiple partitions, and each of the partition is replicated to the 
configured number of replicas. We know that partitions are distributed across the cluster's brokers (nodes). We don't know yet 
how they are distributed and when the partition-to-broker assignment happens. Kafka has two types of brokers: a regular one and 
a controller. There regular broker hosts partition replicas, serving client's requests (just like produce/fetch). The 
controller on the other hand manages the cluster. At this moment it is important to know that: **every time you execute the 
topic creation command, the request is directed to the controller and the controller assigns partitions of a topic to the 
specific brokers. The controller chooses a leader partition too**. 

So how the controller assigns partitions to the brokers ? The actual algorithm takes into consideration both number of 
brokers and the racks where there are situated in a server room. For us, we can assume that we 
have all brokers on the same rack. Kafka controller tries to distribute partitions evenly so each of the partition is on 
a different broker. That's why it spreads partitions using round-robin algorithm, and for each next partition it offsets starting broker, 
so each leader is on a different broker. If you're curious of the details you can read it [here]((https://github.com/apache/kafka/blob/5552f5c26df4eb07b2d6ee218e4a29e4ca790d5c/metadata/src/main/java/org/apache/kafka/metadata/placement/StripedReplicaPlacer.java#L72)).

Let's say we want to create a `test` topic with 3 partitions, each replicated across 3 replicas. 
We have also 3 brokers with ids: 1, 2, 3.

```postgresql
             ----------------- The first node in assignment is a leader. The placement assigns brokers to parititions 
             |                 using round-robin. Leaders offsets by one for each new partition to avoid choosing
             |                 the same broker for each partiton's leader.
             |         
partition 1: 1, 2, 3
partition 2: 2, 3, 1
partition 3: 3, 1, 2
```

How it does look like in reality ? Well, almost the same :)
```postgresql
./kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --replication-factor 3 --partitions 3
Created topic test.

./kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092

Topic: test     TopicId: ha0jRlepRvasAGLxmyx60A PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
```

### Cluster Metadata

We've discovered the topic structure and how the partitions are assigned to the different brokers. The next puzzle 
is how the clients know where to send and fetch messages from. As we've mentioned `ProduceRequest` and `FetchRequests` 
goes to the partitions leaders. To find them, clients send `MetadataRequest` to any broker in a cluster. Each broker 
keeps cluster-wide metadata information on disk. The usual brokers learn about metadata changes from... Kafka controller. 
So every admin change e.g. increasing the number of partitions in a topic goes to the controller. The controller 
propagates the change to the usual brokers via metadata fetch API, and then clients fetch metadata from the brokers.

// P3 

On the picture above, the brokers use `FetchRequests` to get updates about metadata. Internally, metadata is a topic and 
brokers use the existing API between them to fetch the newest metadata information. Producers and consumers on the other
hand, use dedicated `MetadataRequest` for receiving metadata information. They request information about interested
topics, their partition and leaders. Details can be found [here](https://kafka.apache.org/protocol.html#The_Messages_Metadata)
(scroll to the latest version). 

### Replication from leader

As I mentioned before, the follower replicas use the `FetchRequest` for requesting messages from leader. For now, just
assume that the only parameter is offset. So, `FetchRequest(offset=x)` means that:
1. the requesting replica wants messages with offsets `>= x`
2. the requesting replica confirms that it persisted locally all messages with offsets `< x` (and now it wants more)
3. the leader replica seeing that request returns messages with offset `>= x` and save in its internal state that the
requesting replica has all messages with offsets `< x`.

The end of the broker's log for a particular partition is called `LEO` (Log End Offset). It indicates, that broker/replica has
messages with offsets up to `LEO - 1`. So `LEO` is the first offset that the replica doesn't have yet. For example, if the 
replica has log of messages with offsets `[0,1,2,3,4,5]`, its `LEO=6`.

### Replicas vs In-Sync Replicas
As I explained previously, once the `KafkaProducer` sends the `ProduceRequest` to the partition leader, the broker saves 
the message, and then follower replicas keep up by fetching it using `FetchRequest` api. Let's take as an example the 
previously created topic with 3 partitions. For partition `0` there are 3 replicas `Replicas: 1,2,3`. The leader is `1`, the
followers are `2,3`. Each of the follower has different pace of replication. They are independent nodes. While replicating, 
the leader tracks each follower's replication progress, so it can assess which of them are fast and which can't keep up. 
To recall, the topic partition is a log of records, and each record has its offset.

![partition-with-offsets.png]({{site.baseurl}}/img/replication/partition-with-offsets.png)

The leader tracks each of the follower's fetching offset.  


![leader-tracking-followers.png]({{site.baseurl}}/img/replication/leader-tracking-followers.png)

We can now comprehend the concept Kafka heavily depends on to ensure safety and correctness of replication: `in-sync-replicas` (ISR).
For each topic partition, Kafka maintains a curated set of nodes which are in-sync. At a high level, these are the **healthy** brokers 
that **can keep up with fetching the newest messages** from leader. The [documentation](https://kafka.apache.org/documentation/#replication)
says that to be in-sync:

```postgresql
1. Brokers must maintain an active session with the controller in order to receive regular metadata updates.
2. Brokers acting as followers must replicate the writes from the leader and not fall "too far" behind.
```

And there are two broker configs for these conditions:
1. [broker.session.timeout.ms](https://kafka.apache.org/documentation/#brokerconfigs_broker.session.timeout.ms) -
   Remember the controller right ? The simple brokers inform the controller that they are alive via heartbeat requests. If
   the time configured as this config passed between two consecutive heartbeat requests, the broker is not in-sync
   anymore.
2. [replica.lag.time.max.ms](https://kafka.apache.org/documentation/#brokerconfigs_replica.lag.time.max.ms) -
   Each time the same broker fetches messages from specific topic partition's leader, and it got to the end of the log,
   the leader records the timestamp when that happened. If the time that passed between **current time** and the **last fetch 
   which got to the end of the log** is greater than `replica.lag.time.max.ms` the replica hosted by the broker is thrown out 
   of in-sync replicas set. 

![leader-isr-state.png]({{site.baseurl}}/img/replication/leader-isr-state.png)

The second timeout is a little bit harder to understand. The picture above shows that replicas R2 and R3 fetch from different 
positions from leader's log. The `Partition state` is a global view of the partition in a Kafka cluster. 
The `Leader state` is calculated locally by the partition leader. Each time the replica got to the end of the log 
(`FetchRequest{offset=100}` in this case) the leader sets the `lastCaughtUpTimeMs` timestamp for that replica. 
Then, the leader periodically checks if each replica fulfills IRS condition:
`now() - lastCaughtUpTimeMs <= replica.lag.time.max.ms`. If not, that replica is thrown out of the ISR. For now, just assume 
that the leader somehow changes global Kafka view of that partition. So for example if replica `R2` fall out of IRS the 
global view of the partition would be like: `Replicas: [R1, R2, R2], Leader: R1, ISR = [R1, R3]`.

### Consumer's message visibility, watermark and committed messages.

The question that comes to mind when thinking about messages replication is: when the consumers can see the published message ?
when all in-sync replicas replicate the published message. Let's investigate what that exactly means

// P5

The picture above contains a topic with replication factor 3. The producer published 5 messages with offsets: `0,1,2,3,4` 
(remember that offsets starts from 0). The messages `0,1,2` were already replicated by all current in-sync replicas. The 
messages `3,4` were replicated only by one follower, so 2/3 of in-sync replicas has the copy. The offset of the message, 
below which all messages were replicated by all in-sync replicas is called **high watermark**. Everything below that threshold 
is visible to the consumers. So if the high watermark is at offset `X`, all messages visible to the consumers are `<= X-1`. 
That part of the log is also called the **committed log**. Consider the high watermark as an indicator of the safely 
stored part of the log. This is an important thing to understand how Kafka provides high reliability in different failure 
scenarios. We'll get to that.

Let's find out how and when exactly the high watermark is moved forward. 

We have the following setup:

```postgresql
TopicA - partitions: 1 (one partition P1 for simplicity), replication factor: 3 (leader and two followers)
leader = R1
replicas = [R1, R2, R3]
ISR (in-sync replicas) = [R1, R2, R3]
```

I'll use the marker `T[X]` for designating the time progression. Remember that `LEO` is the log end offset, and it is the next offset 
that the replica doesn't have yet for partition `P`. The leader moves the high watermark, and it's tracking each
of the followers `LEO`. I purposely show `LEO` and watermark only on a leader. In reality, each of the replica tracks 
its own `LEO` and high watermark, but I'll omit it now for simplicity. 

```postgresql
----------------------------------T1--------------------------------------------
KafkaProducer publishes 3 messages with offsets 0,1,2 to the leader R1.

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = []
R3          | log = []
----------------------------------T2--------------------------------------------
Replica R2 sends FetchRequest(offset=0) to the leader, the leader responds with messages it has in log. Note that 
R2 LEO didn't change as the leader has not yet been confirmed about persisting the messages by the follower. It has to wait for 
the next FetchRequest(offset=3) to be sure that previous offsets were saved. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = []
----------------------------------T3--------------------------------------------
The replica R3 sends FetchRequest(offset=0) to the leader, the leader responds with messages it has in log. The same 
situation as above. The replica didn't confirm getting the messages yet. The high watermark cannot be moved forward. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 0, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T4--------------------------------------------
KafkaConsumer (client app) sends FetchRequest(offset=0) to the leader. The leader doesn't respond with any messages 
because the high watermark is 0.
----------------------------------T5--------------------------------------------
The replica R2 sends FetchRequest(offset=3) to the leader, the leader has no messages with offset >= 3, so it doesn't return any  
messages. But now, the leader knows that replica R2 persisted messages with offset < 3. The R2 LEO changes, and 
points to the next message offset not present in the R2 log. The high watermark cannot be 
moved forward as there is still one in-sync replica R3, which didn't confirm getting the message. 

R1 (leader) | log = [0,1,2], HighWatermark = 0, R1 LEO = 3, R2 LEO = 3, R3 LEO = 0
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T6--------------------------------------------
The replica R3 sends FetchRequest(offset=3) to the leader, the leader has no messages with offset >= 3, so it doesn't return any  
messages. But now, the leader knows that replica R3 persisted messages with offset < 3. The R3 LEO changes to 3. 
All in-sync replicas now reached the offsets 3: have their LEO=3. The high watermark can move forward, because all in-sync replicas confirmed 
getting messages with offset < 3. To be more precise: the high watermark can progress when all in-sync replicas' LEO exceeds current  
watermark. Because current HighWatermark = 0 and R1 LEO = 3, R2 LEO = 3, R3 LEO = 3, the leader can set watermark to 3 as well.
The [0,1,2] is now committed log and is visible to KafkaConsumer clients. 

R1 (leader) | log = [0,1,2], HighWatermark = 3, R1 LEO = 3, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]

----------------------------------T7--------------------------------------------
KafkaProducer publishes 2 messages with offsets 3,4 to the leader R1.

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 5, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T8--------------------------------------------
The replica R2 sends FetchRequest(offset=3) to the leader, the leader responds with 2 messages [3,4] with offset >= 3. 
R2 LEO does not change on the leader, because it lacks of the message persistence confirmation - the leader has to 
wait for FetchRequest(offset=5) from R2 to confirm saving new messages. 
The high watermark does not move forward. It can only progress when all in-sync replicas' LEO exceeds the current 
watermark. 

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 3, R2 LEO = 3, R3 LEO = 3
R2          | log = [0,1,2,3,4]
R3          | log = [0,1,2]
----------------------------------T9--------------------------------------------
The replica R2 sends FetchRequest(offset=5) to the leader, the leader has not any messages with offset >= 5. The leader 
changes its state for R2 LEO to 5 as it got confirmation for saving messages with offset < 5. The high watermark does not 
change as there is still a replica R3, which is in-sync, and its LEO <= high watermark. 

R1 (leader) | log = [0,1,2,3,4], HighWatermark = 3, R1 LEO = 5, R2 LEO = 5, R3 LEO = 3
R2          | log = [0,1,2]
R3          | log = [0,1,2]
----------------------------------T10--------------------------------------------
KafkaConsumer (client app) sends FetchRequest(offset=0) to the leader. The leader responds with messages having 
offset < high watermark: [0,1,2] which is a committed log.
```

### Shrinking ISR

We said that for advancing the watermark, all in-sync replicas have to replicate the message. But what happens if one of 
the replica becomes unavailable or slow ? Effectively, it can't keep up with a replication. Without a special treatment 
we would end up with an unavailable partition - the HW can't progress and consumers can't see new messages. Obviously we 
can't just wait for a replica until its recovery, this is not something a highly-available system would do. 
Kafka handles such a situation by shrinking the ISR. You remember the two [conditions](#replicas-vs-in-sync-replicas) 
for a replica to stay in-sync, right ? I didn't mention what exactly happens. There are basically two scenarios when
the conditions stop applying:

1. The unhealthy replica stops sending heartbeats to the controller. The controller fences a replica. That means, it 
is removed from the ISR of all its topic partitions. This implies losing leadership too. If the removed replica is a leader 
for any partitions, the controller performs leader election and pick one of the broker from the new ISR as a new leader.
That information is disseminated across cluster metadata (from controller -> to other brokers -> and to clients).

// P7
// P8

2. The second party allowed to modify the partition's ISR (besides the controller) is a partition leader. 
A replica that does not fetch data at all or fetches it too slow, is removed from ISR by a leader of that partition. The 
leader sends an `AlterPartition` request to the controller with the new ISR. The controller persists that information 
in metadata which is then spread across the cluster. 

// P9: TODO

// TODO: 

###  Min-ISR

### Preferred leader and leader failover

Each Kafka topic partition has a leader replica chosen by a controller during topic creation. This is called a preferred leader 
replica. I previously explained how Kafka assigns leaders to brokers. leadership can change.


// TODO: Partitions vs availability - does the completely failed partition appears in producer metadata ?  -> yes
// TODO: what about ELR ? maybe just add information at the end 
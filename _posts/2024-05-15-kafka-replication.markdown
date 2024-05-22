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
```shell
./kafka-topics.sh --create --topic A --bootstrap-server localhost:9092 --replication-factor 2 --partitions 2
```

A command to see the topic details:
```shell
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

```
             ----------------- The first node in assignment is a leader. The placement assigns brokers to parititions 
             |                 using round-robin. Leaders offsets by one for each new partition to avoid choosing
             |                 the same broker for each partiton's leader.
             |         
partition 1: 1, 2, 3
partition 2: 2, 3, 1
partition 3: 3, 1, 2
```

How it does look like in reality ? Well, almost the same :)
```
./kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --replication-factor 3 --partitions 3
Created topic test.

./kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092

Topic: test     TopicId: ha0jRlepRvasAGLxmyx60A PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
```

// TODO: Partitions vs availability - does the completely failed partition appears in producer metadata ?  -> yes

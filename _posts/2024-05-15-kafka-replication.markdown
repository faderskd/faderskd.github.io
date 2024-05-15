---
layout: post
title:  "Kafka replication deep dive"
date:   2024-05-15
---

### Replication

As any serious distributed storage, Kafka provides a numerous ways for configuration of a durability and availability. 
You've heard it a million times, but I'll repeat it the (million + 1) time: in a distributed system any node will 
inevitably fail. Even if the hardware won't break, the power outage, OS restart, network slowdown can take a node 
down or partition it from the rest of the system. Overcoming those issues in Kafka requires understanding different 
tunables. Every configuration brings a tradeoff, every tradeoff requires awareness of the consequences. 
Hence, our objective is to gain the essential internals of Kafka replication. This knowledge will enable us to set up 
the system in a more informed and confident manner. 

### Topics, Partitions, Brokers

Kafka divides a messages published to a set of topics. A topic is just one category of related messages. It's something 
like SQL table or database collection. The topic divides into partitions. A partition 

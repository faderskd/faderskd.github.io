---
layout: post
title: "Kafka producer deep dive"
date: 2024-10-14
---

> [INFO]  
> The post covers replication KafkaProducer version (>=3.7).
> {: .block-tip }

### KafkaProducer send

1. Call interceptors on a record - this can change the record before it will be serialized.
2. Fetch metadata for the topic of the record. If no metadata is available for topic, the producer will add
   the topic to the metadata list and schedule metadata update. The call is blocked until metadata is available. 
   Note that matatada is updated only for interested topics - the ones for which the producer has records to send.
   #TODO - how metadata updating works
3. Serialize the record key and value.
4. Calculate the partition for the record: 
   1. If partition is specified in the record, use it.
   2. If custom partitioner is specified, use it.
   3. If key is present, use the hash of the key.
   4. If key is not present, use return UNKNOWN_PARTITION and delay partition choice.
5. Append record to accumulator:
   1. Get current sticky partition. 
   2. If we don't have load stats
      1. get random available partitions from the metadata (point 2). Available partition is 
      a one that has a leader. 
      2. If we don't have available partitions just get random partition.
   3. If we have load stats, get random partition using probability proportional to the lowest load: the lower the load
      the higher the probability of choosing the partition. The load stats are calculated using cumulative frequency table. 
      It works as follows:
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
   6. How accumulator informs partitioner to switch to another partition?
6. Once message is appended to the batch, we return a future to the user. The future is completed when the record is sent. 
   If the batch is full we notify the sender so it can start sending the batch to the broker. 
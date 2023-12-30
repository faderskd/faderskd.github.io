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
Let's say we have an online shop app, with three Kafka topics: `purchases`, `stock`, `parcels`. Whenever the customer buys
a product, the `purchase` event is produced. It is then processed and two other events are produced:
`stock` and `parcel`. The former indicates difference in stock for a product after purchase, the latter notifies the courier 
that it has a new package to pick up.  

![three-topics.png](/img/transactions/three-topics.png)

Obviously we don't want the situation where we send a parcel but don't decrease the product stock. We don't want the parcel
to be sent twice too. The `stock` and `parcel` events should be published together (as a unit) or none of them.

# Transactions usage from client perspective
Before we dig dive into internals of transactions let's use them in practice. 
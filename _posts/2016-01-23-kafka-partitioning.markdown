---
layout: post
title:  "Kafka's DefaultPartitioner and byte arrays"
date:   2016-01-23 00:00:00
categories: kafka big data partitioner java scala byte array
---

Apache Kafka is a simple, horizontally-scalable durable message queue. The fundamental unit of scale in a Kafka cluster is a partition: a partition is a single log, which resides on a single disk on a single machine (it may be replicated). If you aren't familiar with the concept of logs, partitioning or how Kafka scales horizontally, Jay Kreps has the canonical post on Kafka design here: [The Log](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying). The [Kafka documentation](http://kafka.apache.org/documentation.html) is also generally very good.

One important property of Kafka is that the order of messages within a single partition is guaranteed, and the order of messages across multiple partitions is undefined. Different consumers, or the same consumer at different times, can interleave messages from different partitions in a way that isn't deterministic. If you need to make sure a set of messages arrive in order, relative to each other, you need to make sure they're all routed to the same partition.

Kafka's documentation makes it seem really clear how to route messages to specific partitions: you set a key. From the docs:

>The partition API uses the key and the number of available broker partitions to return a partition id. This id is used as an index into a sorted list of broker_ids and partitions to pick a broker partition for the producer request. The default partitioning strategy is hash(key)%numPartitions. If the key is null, then a random broker partition is picked. A custom partitioning strategy can also be plugged in using the partitioner.class config parameter.

It's important to remember here: the producer is solely responsible for choosing where messages go. A broker will accept any messages it receives, it won't enforce any partitioning scheme for you. You could (deliberately or otherwise) have different producers with different partitioning schemes, producing into the same partitions. 

Kafka actually has two producer APIs - the new, all-Java one, and the old, Scala one. The Scala one is not recommended for new projects, but it's still widely used in legacy codebases. The new producer (`org.apache.kafka.clients.producer.KafkaProducer`), uses `org.apache.kafka.clients.producer.internal.Partitioner` as the default partitioner. The partitioning method:

```
return Utils.abs(Utils.murmur2(record.key())) % numPartitions;
``` 

That seems totally reasonable. `record.key()` is a byte array, `Utils.murmur2` is an implementation of MurmurHash. Everything checks out, two events with the same key are guaranteed to be sent to the same partition. This is the property we want.

 What about the old API? `kafka.producer.Producer` uses `kafka.producer.DefaultPartitioner` as the default partitioner. How does the partitioning method work? A little bit of Scala:

```
Utils.abs(key.hashCode) % numPartitions
```

Whoops! `hashCode` is OK for some key types, but it doesn't work at all if your key is a `byte[]` (or any other array) - the hash will be based on the address of the array, not the contents. In other words, given two `byte[]` keys with identical contents, they'll hash differently and be routed to different partitions. The solution to this is really simple: use the `kafka.producer.ByteArrayPartitioner` instead. The `partition` method there looks like:

```
Utils.abs(java.util.Arrays.hashCode(key.asInstanceOf[Array[Byte]])) % numPartitions
``` 

Much better. Cast the key (which has the Java type `Object`) to a `byte[]`, and use `Arrays.hashCode` to hash the contents.

In conclusion, if you're using the old Scala API and your keys are byte arrays, events may not be partitioned the way you expect. You can configure the producer to use the `ByteArrayPartitioner` by setting the `partitioner.class` property in the producer configuration to `kafka.producer.ByteArrayPartitioner`. The new Java producer doesn't have this unexpected behaviour, it works as expected with the default partitioner.

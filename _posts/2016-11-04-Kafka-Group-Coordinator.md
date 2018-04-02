---
layout: post
title: Offset topic and consumer group coordinator of Kafka
excerpt: An attempted understanding from source code and limited documentation
category: kafka
---

This article is to discuss two subjects
that are not frequently or clearly covered by official document or
online sources.

#### Offset topic (the `__consumer_offsets` topic)

It is the only mysterious topic in Kafka log
and it cannot be deleted by using
[TopicCommand](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/admin/TopicCommand.scala).
Unfortunately there is no dedicated official documentation
to explain this internal topic.
The closest source I can find is
[this set of slides](http://www.slideshare.net/jjkoshy/offset-management-in-kafka).
I strongly recommend reading it if you wish to understand
how this internal topic works.

In short,
`__consumer_offsets` is used to store *offsets* of consumers
which was previously stored only in ZooKeeper before version 0.8.1.1.
At the latest version of 0.8.X serious, i.e. 0.8.2.2,
the storage location of offsets can be configured by
`offsets.storage` whose value can be either `kafka` or `zookeeper`.
If it is `kafka`,
consumers are still able to commit offsets to ZooKeeper
by enabling `dual.commit.enabled`.
However since version 0.9,
consumer offsets have been designed to be stored on brokers only.

The partition key of the messages in `__consumer_offsets`
was handled in
[OffsetManager](https://github.com/apache/kafka/blob/0.8.2/core/src/main/scala/kafka/server/OffsetManager.scala)
at version 0.8.X
and has been named as `OffsetKey` in
[GroupMetadataManager](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/coordinator/GroupMetadataManager.scala)
since version 0.9.0.
It contains three pieces of information: groupId, topic and partition number,
and the key is serialized/de-serialized according to a schema called
`OFFSET_COMMIT_KEY_SCHEMA`.
The usage of schema is primarily for backward compatibility.
Unlike the behavior of
[DefaultPartitioner](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java),
the partition number inside `__consumer_offsets` is *not*
determined by the hash value of partition key
but only determined by the hash value of consumer group,
which is as simple as
```
Utils.abs(groupId.hashCode) % numPartitions
```
Here the `numPartitions` is configured by the value of
`offsets.topic.num.partitions` in broker configs
and is 50 by default.
This algorithm will be re-introduced later
in the part of consumer group coordination.

The values of offset messages which is named
[OffsetMetadata](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/common/OffsetMetadataAndError.scala)
have two versions.
At version 0.8.X, the value contains offset, metadata (often empty)
and a timestamp
while the timestamp had been split into commit and expire timestamps
since version 0.9.0.
Console consumers are able to consume messages from internal topics
and print them out nicely.
A sample command has been shown below:

```
./kafka-simple-consumer-shell.sh --topic __consumer_offsets \
--partition 49 \
--broker-list localhost:9092 \
--formatter "kafka.server.OffsetManager\$OffsetsMessageFormatter"
```
By the way, `absolute("testGroup".hashCode() % 50) = 49`
which is the reason why partition 49 was specified.
It prints out:

```
[testGroup,testTopic-development,0]::OffsetAndMetadata[11,NO_METADATA,1478243992053]
[testGroup,testTopic-development,0]::OffsetAndMetadata[12,NO_METADATA,1478243992086]
[testGroup,testTopic-development,0]::OffsetAndMetadata[13,NO_METADATA,1478243992096]
[testGroup,testTopic-development,0]::OffsetAndMetadata[14,NO_METADATA,1478243992110]
```

However this is not the end of story,
because `__consumer_offsets` is also used by group coordinator
to store group metadata!
The following section discusses another new feature
introduced since version 0.9.0.

#### Group coordinator (coordinated rebalance)

This section is my humble and shallow understanding about
broker coordinator of consumer groups.
Correct me if I ever miss something or make any mistake.

The introduction of coordinator, according to the
[official wiki of Kafka](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design),
is to solve the *split brain* problem
which is a well known problem in a distributed system.

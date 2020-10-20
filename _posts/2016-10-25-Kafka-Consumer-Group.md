---
layout: post
title: Thoughts on Consumer Group of Apache Kafka
description: Simple experiences from the usage of consumer group in production
categories: kafka
---

#### Introduction

The concept of consumer group is intriguring.
According to the
[official documentation of Kafka](http://kafka.apache.org/documentation.html#introduction),
one of its main objectives is to achieve message multicast and broadcast.
Often it is not an easy concept to grasp at the very beginning.
This article will share a few issues and discoveries about consumer group.

A good way to start understanding consumer group is
looking at its data structure within ZooKeeper,
which is illustrated below.

![Your browser does not support img](/assets/images/consumer-group-zk.png){: .responsive-img }

You may see each consumer group as a separated world in terms of message consumption.
Each consuemr group maintains its own consumer ID's,
partition owners and offsets.

#### Rebalance

Consumer rebalance happens when consumers join or leave a consumer group or
when the topics within a consumer group have new partitions.

Consumer ID uniquely identitfies a consumer.
It is generated automatically when a consumer is launched.
The source code of its generation is defined in [ZookeeperConsumerConnector](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala)
and shown below.

{% highlight scala %}
var consumerUuid : String = null
config.consumerId match {
  case Some(consumerId) // for testing only
  => consumerUuid = consumerId
  case None // generate unique consumerId automatically
  => val uuid = UUID.randomUUID()
  consumerUuid = "%s-%d-%s".format(
    InetAddress.getLocalHost.getHostName, System.currentTimeMillis,
    uuid.getMostSignificantBits().toHexString.substring(0,8))
}
config.groupId + "_" + consumerUuid
{% endhighlight %}

If you go into `/owners` directory and check the owner of a partition,
you should find a very similar value to consumer ID.
The owner of a topic-partition is defined in the form of `[consumerId]-[threadId]` in which `threadId` is used
because one partition is meant to be consumed by exactly one thread.
In many cases, the `threadId` is equal to the partition number which
the consumer thread owns.

A consumer group is *well balanced*
if each partition inside it is owned by exactly one consumer thread.
You may use the instructions from [this link](https://cwiki.apache.org/confluence/display/KAFKA/System+Tools#SystemTools-VerifyConsumerRebalance)
to verify the result of consumer rebalance.
This tool should work well because I have fixed a bug in `VerifyConsumerRebalance.scala` in [this pull request](https://github.com/apache/kafka/pull/1612).

Since version 0.9.0,
Kafka has used *brokers* to coordinate the rebalance process of consumers.
I also have thought about it and then wrote this blog.

#### Broadcast

Broadcast by Kafka is relatively cheap:
you just need to put each consumer in different consumer groups,
such that the offset of each consumer is different.

Once I read some articles online which suggest use `UUID` within
a consumer group, for example [this question](http://stackoverflow.com/questions/30647544/kafka-multiple-consumers-for-a-partition) from stackoverflow and
it has received more than 5 upvotes.
Unfortunately I **strongly discourage** such usage.

Firstly and most importantly, consumer groups are stored as persistent nodes in ZooKeeper.
Often consumers need to be shutdown and started,
which may not be frequent in commercial environments
but is expected to be quite frequent in develop, test and staging phases.
As time goes by a huge number of consumer groups nodes
shall be accumulated in `/consumers` node and
most of them are not in use.
A direct result is
it is almost impossible to list out all the consumer groups.
ZooKeeper client throws runtime exception because the buffer overflows due to
the overwhelming amount of child nodes.
The error message is something like:

```
IOException Packet <len12343123123> is out of range
```

Even you configure the `jute.maxbuffer` parameter as some blogs suggest,
the chance of receiving a response
before your patience runs out is extremely low.
I have encountered a situation where `/consumers` has more than 90,000
child nodes and was unable to list up consumer groups ever since.
Another direct result is that some features of [ConsumerGroupCommand](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/admin/ConsumerGroupCommand.scala)
(used in `kafka-consumer-groups.sh`)
will fail because internally it lists out all consumer groups.

However, the huge number of consumer groups should *not* be a problem
for the normal operation of Kafka,
because the brokers do not need to know all the groups.
Even though from version 0.9 onward,
consumer rebalance is coordinated by brokers,
brokers do not scan groups when deciding which broker coordinates
which subset of them.
They only manage those consumer which join or leave them.
As far as what I have read from [GroupCoordinator](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/coordinator/GroupCoordinator.scala),
brokers do not scan all consumer groups.

Next, `UUID` is hardly queryable.
Its value is generated at runtime and unpredictable.
If you wish to describe a consumer group or check its offsets,
you will face problems on finding the exact value of it.
A workaround is to *log* its value somewhere.

Last but not the least,
group names with `UUID` in them is brand new every time.
Upon the creation of a new consumer group,
the value of offsets is determined by offset reset,
in other words the setting of `auto.offset.reset`.
No matter the value is `smallest` or `largest`,
the consumer either consumes many messages repeatedly or
skip some messages.

#### Delete a group

Deleting a consumer group from ZooKeeper is
as easy as one single instruction.
However, it implies that the offset information
will be permantly deleted from ZooKeeper.
Do it only when you are 100% sure that
the target group will never be reused.

From version 0.9.0 onward,
[ConsumerGroupCommand](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/admin/ConsumerGroupCommand.scala)
introduces a feature to remove consumer groups.
Nevertheless, it prohibits the removal of *active* consumer group by
checking the number of children under consuemr registry dicrectory
(i.e., `/consumers/[group_name]/ids`).
When you wish to remove a consumer group by a ZooKeeper client,
please take notice of active consumers as well.

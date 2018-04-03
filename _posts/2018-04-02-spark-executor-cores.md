---
layout: post
title: Spark's number of cores rethought
excerpt: From an infrastructural perspective, what do the number of executors and cores actually mean?
category: spark
---

There are tons of articles talking about setting the number of executors and cores for [Spark](http://spark.apache.org/) applications.
Hence I am _not_ going to discuss anything about tuning these 2 parameters.
Instead I would like to address what does the values of these parameters mean
for nodes or containers which are executing your tasks.

### Summary

The number of cores has no direct relation to physical CPU cores,
instead it is a logical counter
which determines the number of concurrent
tasks that are able to run on one executor.
I believe that many articles you can find have already addressed it.
However from the perspective of servers or containers
that receive tasks from the driver process,
the number of cores is able to limit the total number of tasks running
on one server/node/container.

### Standalone mode

Standalone mode is very easy to setup but has few features in
scheduling and access control.
Though I have touched both standalone and YARN mode,
starting from standalone mode is a good choice
because the implementation details are easy to inspect, compared to YARN mode
where lots of details are handled by YARN.

Let's start from worker process first.
It is recommended that each server only runs one single worker process
and I am going to treat it as an assumption.

What are workers and executors?

Each worker is Java process and so is an executor.
If the number of cores is not specified when starting a worker,
the worker will pick the number of physical cores from OS
to be its capacity of cores.
For Spark applications that demand executors,
worker processes spawn executor processes as responses to such requests.
Upon each creation of an executor process,
worker deduct the amount of memory and cores from the total value
it controls.
In [Worker]()
class, you can find in method `launchExecutor` that
for each launch of a new executor, the count of used cores is incremented.
Therefore as I said,
the number of memory and cores are more like
_logical_ values rather than the actual amount of resources each executor uses.

What does it look like in deeper details?
The number of memory (controlled by `spark.executor.memory`)
determines the **maximum heap size** of executor process,
by forming the `-Xmx` parameter.
For example if you pass in `--conf spark.executor.memory=10240m`
when submitting an Spark application,
the executor Java process will include a command line argument as `-Xmx 10240m`.

[ExecutorRunner](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/deploy/worker/ExecutorRunner.scala)
is the wrapper class around each executor process.
It has an attribute which is an instance of Java
[Process](https://docs.oracle.com/javase/8/docs/api/java/lang/Process.html).
Actually [CoarseGrainedExecutorBackend](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/deploy/worker/Worker.scala)
is the main entry point class of executor process
which is able to be observed by using `ps -ef` command.
As you dig deeper in the source code,
you should be able to find that each task is a thread submitted
to a thread pool.
`spark.executor.cores` thus controls the upper limit of concurrent tasks
running in one executor process.
Their resources are shared within one process.
As quite a number of online sources have pointed out,
a big number of cores, say 8,
causes the overhead of context switch to be big and
actually slow down the overall performance.
A number between 2 and 4 (inclusive) is recommended for most Spark applications.

### YARN mode

YARN provides a much richer set of features such as
(very fine-grained) capacity scheduling, label-based scheduling and access control.

The table below helps you compare and understand standalone and YARN mode
side by side.

|  YARN | standalone |
|---------------------|--------|
| ResourceManager     | Master  |
| NodeManager         | Worker  |
| yarn.nodemanager.resource.memory-mb  | --memory |
| yarn.nodemanager.resource.cpu-vcores | --cores  |

A major difference between YARN and standalone mode in terms of resource control
is that workers stop spawning new executors
when either of the resources is exhausted.
However the `DefaultResourceCalculator` only uses _memory_ to control
the resources used by executors.
As a result sometimes you can see that the available of vcores on a node
become negative.
`DominantResourceCalculator` behaves the same way as workers in standalone mode.
It chooses the dominant resource as the upper limit for resource usage.


### An infra perspective

I have heard that some companies use Kubernetes to spawn containers to host NodeManagers,
when there are huge demands for resources.

An ideal scenario is where the memory and cores in your cluster
are consumed at the same pace.
When memory is used up,
cores should be exhausted as well.
To achieve better utility of your Hadoop slaves,
tune these parameters such that
when memory and cores are used up by executors,
memory in OS is close to fully utilized and CPU load is slightly below
maximum capacity.
This advice is given in condition that Spark executors are the dominant
processes running on your servers
and you should always leave some memory and computational power for
other processes than Spark executors and tasks.
Your Spark applications might have very strange behavior,
sometimes even failures,
when the CPU load of Hadoop slaves is extremely high.

If you have both IO intensive (such as ETL)
and computationally intensive applications (such as data science apps),
consider introducing label-based scheduling before tuning.

### Epilogue: data engineering in Shopee

As the primary data provider in a leading E-commerce platform operating across Southeast Asia,
data engineering team is able to handle TB-level in one ETL flow and
this number is still increasing fast.
We run Spark jobs on top of an in-house Hadoop cluster
whose size is among the top-tier in Singapore as well as SEA region, hopefully.

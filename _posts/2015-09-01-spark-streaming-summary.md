---
layout: post
title: "Summary of Spark Streaming"
date: 2015-09-01
author: "Jerry Shao"
tags:
    - spark
    - cloud
    - spark streaming
---

This blog is a summary of Spark Streaming, to introduce the basic concepts of Spark Streaming, as well as the internals of Spark Streaming, comparison to Storm, also in-production usage of Spark Streaming and important JIRAs in the history of Spark Streaming evolution.

## What is Spark Streaming ##

Spark Streaming is a component based on Spark for doing large-scale stream processing.

* It Extends Spark for doing large scale stream processing

* Scales to 100s of nodes and achieves second scale latencies

* Efficient and fault-tolerant stateful stream processing

* Simple batch-like API for implementing complex algorithms


### Discretized Stream Processing ###

> Run a Streaming computation as a **series of very small, deterministic batch jobs**.

Spark Streaming chops the live stream in to batches of X seconds, Spark treats each batch of data as RDDs and process them using RDD operations.

Essentially Spark Streaming is **micro-batching** system, by chopping the live stream into a series of very small batches, it could achieve the similar behavior as real-time streaming framework. You could simply simulate the behavior of Spark Streaming by writing a Spark batch job to process a mount of live data, and adding this job to a crontab to get called at a period of time.

<img src="http://spark.apache.org/docs/latest/img/streaming-flow.png" alt="spark streaming" width="640">

### DStream ###

Similar to RDD on Spark, Spark Streaming provides a high-level abstraction called **DStream**, which represents a continuous stream of data. Internally DStream is represented as a sequence of RDDs.

Like RDD, DStream can be got from input dstream like Kafka, Flume and so on, also transformation could be applied on the existing DStream to get a new DStream.

Here is the code snippet of streaming word count:

    val lines = KafkaUtils.createStream(ssc, zkQuorum, group, topicMap).map(_._2)
    val words = lines.flatMap(_.split(" "))
        val wordCounts = words.map(x => (x, 1L))
      .reduceByKeyAndWindow(_ + _, _ - _, Minutes(10), Seconds(2), 2)
    wordCounts.print()

1. Get the input DStream from Kafka input source.
2. Apply high-level transformations on the input DStream to get the new DStreams.
3. Trigger the action `print()` on the DStream to finalize this job DAG.

So basically Spark Streaming's programming pattern is quite similar to Spark, so it is simple and low learning curve for users to get started with Spark Streaming.

The basics of Spark Streaming can be found in [**Streaming doc**](http://spark.apache.org/docs/latest/streaming-programming-guide.html) and [**Ampcamp traning materials**](http://ampcamp.berkeley.edu/wp-content/uploads/2013/07/Spark-Streaming-AMPCamp-3.pptx)

### What Additionally Spark Streaming Provides ###

So you may wonder why not directly write a Spark program and get called periodically to simulate streaming program? Yes you could, but you should also handle some problems where Spark Streaming already have done for you.

1. Spark Streaming offers various of connectors like Kafka, Flume, Kinesis and so on, it is necessary for a streaming program and Spark Streaming is well supported.
2. Fault tolerance. Fault tolerance is quite important for a distributed program. Spark Streaming provides several ways to guarantee this: two copies of received data, WAL mechanism, metadata checkpoint and recovery.
3. Instrumentation. Spark Streaming provides several instrumentation tools for you to better know the run-time internals of Spark Streaming.

There are some materials for you to better understand the internals of Spark Streaming:

* [**Deep dive into Spark Streaming**](http://www.slideshare.net/spark-project/deep-divewithsparkstreaming-tathagatadassparkmeetup20130617?qid=f875139e-920b-47bb-88f4-67cc3554858e&v=default&b=&from_search=6)
* [**Improved Fault-tolerance and Zero Data Loss in Spark Streaming**](https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html)
* [**Improvements to Kafka integration of Spark Streaming**](https://databricks.com/blog/2015/03/30/improvements-to-kafka-integration-of-spark-streaming.html)
* [**Integrating Kafka and Spark Streaming: Code Examples and State of the Game**](http://www.michael-noll.com/blog/2014/10/01/kafka-spark-streaming-integration-example-tutorial/)

### Pros and Cons of Spark Streaming ###

**Pros:**

1. Ease of use, low learning curve, no additional effort when you already understand Spark.
2. Highly integrated in the Spark ecosystem, mutual-operability between different Spark components.
3. High throughput with fast fault recovery.
4. Batch like high-level abstracted API for you to focus on the processing logic.

**Cons:**

1. Micro-batch processing model makes latency relatively higher than other record-based system.
2. Checkpoint mechanism is not so robust and upgradable.


## What's the difference compared to Storm ##

The major difference between Spark Streaming and Storm is the process engine. Spark Streaming uses Spark internally as its process engine, so this restricts Spark Streaming to be a micro-batching model. Whereas Storm is a streaming model, in which data is come and processed as water flow.

This model difference makes Spark Streaming as a micro-batching system and Storm as a event-driven system.

Another difference is fault recovery. Spark Streaming as what I mentioned before is driven by Spark internally, so the fault mitigation is relying on Spark's mechanism like straggler, task rerun. On the other side Storm uses a upstream recovery mechanism, in which it provides a anchoring system to monitor the data loss and notify the upstream**\***, to do fault recovery.

\* [**Guaranteeing Message Processing**](https://storm.apache.org/documentation/Guaranteeing-message-processing.html)

Anchoring system requires more cpu and network resources to track each message, this will effect the total throughput, so generally the throughput of Storm is lower than Spark Streaming.

From user's point, Storm provides low-level primitives, whereas Spark Streaming offers high-level abstractions. On the one side high-level API is easy to use, but low-level primitives offers strong operability.

Here are some materials to compare Spark Streaming with Storm:

* [**Apache Storm and Spark Streaming Compared**](http://www.slideshare.net/ptgoetz/apache-storm-vs-spark-streaming?qid=9570aa50-7fcd-4716-9dc6-b86ba0a01270&v=default&b=&from_search=1)
* [**Apache Storm vs. Spark Streaming – two Stream Processing Platforms compared**](http://www.slideshare.net/gschmutz/apache-stoapache-storm-vs-spark-streaming-two-stream-processing-platforms-comparedrm-vsapachesparkv11?qid=9570aa50-7fcd-4716-9dc6-b86ba0a01270&v=default&b=&from_search=2)

## History and Key Improvements of Spark Streaming ##

Spark Streaming was first brought into Spark at version 0.7 with papers [**Discretized Streams: Fault-Tolerant Streaming Computation at Scale**](http://people.csail.mit.edu/matei/papers/2013/sosp_spark_streaming.pdf). At that time, Spark Streaming is very rudimentary with many functionalities missed, also not so stable to put into in-production use.

[**[SPARK-1332] Improve Spark Streaming's Network Receiver and InputDStream API for future stability**](https://issues.apache.org/jira/browse/SPARK-1332)

In the version 1.0, Spark Streaming refactor the receiver and input stream part to make it more stable and extendable for feature use and user extension.

[**[SPARK-1386] Spark Streaming UI**](https://issues.apache.org/jira/browse/SPARK-1386)

Add the UI for Spark Streaming to better monitoring the running status of Spark Streaming.

[**[SPARK-2377] Create a Python API for Spark Streaming**](https://issues.apache.org/jira/browse/SPARK-2377)

Add the Python API support for Spark Streaming.

[**[SPARK-3129] Prevent data loss in Spark Streaming on driver failure using Write Ahead Logs**](https://issues.apache.org/jira/browse/SPARK-3129)

Spark's implementation makes it suffer from data loss when driver and executor lost. This Patch introduces a write ahead log mechanism to make sure no data lost when driver is failed. The basic implementation is to write data into HDFS as a reliable storage when received from external source.

[**[SPARK-4964] Exactly-once + WAL-free Kafka Support in Spark Streaming**](https://issues.apache.org/jira/browse/SPARK-4964)

Spark Streaming WAL mechanism brought additional overhead for Kafka input stream, since data is persisted in Kafka, no need to put into HDFS again, this patch provides another way of fetching data from Kafka with Spark Streaming.

[**[SPARK-6702] Update the Streaming Tab in Spark UI to show more batch information**](https://issues.apache.org/jira/browse/SPARK-6702)

Provides a better UI visualization for Spark Streaming.

[**[SPARK-7398] Add back-pressure to Spark Streaming**](https://issues.apache.org/jira/browse/SPARK-7398)

Add a back pressure mechanism in Spark Streaming for better flow control.

## Adoption of Spark Streaming ##

Spark Streaming in Netflix: [**Spark Streaming Resiliency**](http://www.slideshare.net/PrasannaPadmanabhan/spark-streaming-resiliency-bay-area-spark-meetup?qid=19f96e48-78d6-4679-842e-4eebbf8af45b&v=qf1&b=&from_search=18).

Spark Streaming in Alibaba: [**Dynamic Community Detection for Large-scale e-Commerce data with Spark Streaming and GraphX**](http://www.slideshare.net/SparkSummit/dynamic-community-detection-ming-huang?qid=19f96e48-78d6-4679-842e-4eebbf8af45b&v=qf1&b=&from_search=21)

...

## Conclusion ##

Compared to the early version of Spark Streaming, its stability and maturity is increased a lot. Now more and more companies use Spark Streaming in their in-production environment as a replacement of Storm in some scenarios.

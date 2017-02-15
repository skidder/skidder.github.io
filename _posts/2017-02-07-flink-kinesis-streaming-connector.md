---
layout: post
title: Building and Using the Flink Kinesis Streaming Connector
---

# What is Kinesis?
[Kinesis](https://aws.amazon.com/kinesis/) is a hosted service from Amazon Web Services that offers the ability to publish & consume durable streams of high-volume data. There's no limit to the number of sources that feed a Kinesis stream, making Kinesis a convenient ingestion point for a data processing pipeline.

Popular uses for Kinesis include:

 * Application log records
 * IoT device measurements
 * Stock quotes
 * Social media posts

Compelling Kinesis features include:

 * Message durability
 * Access to any point in the stream during the last 24-hours (default)
 * Configurable provisioned-throughput on a configurable number of shards

Kinesis is [often compared](https://blog.insightdatascience.com/ingestion-comparison-kafka-vs-kinesis-4c7f5193a7cd#.5mfeavesi) to the Apache Kafka project. Kafka is the [most popular stream source](http://data-artisans.com/flink-user-survey-2016-part-1/) in use with Apache Flink. The second-most popular is HDFS. Kinesis is further down the list, but is gaining popularity among organizations that don't have the desire or technical resources to maintain an Apache Kafka installation.

# How Flink uses Kinesis Streaming Connectors
[Apache Flink](https://flink.apache.org) uses sources and sinks to represent inputs and outputs to a streaming application pipeline. These can includes Apache Kafka, Amazon Kinesis, HDFS, and more. Messages read from Kafka or Kinesis are indistinguishable as they flow through the application pipeline. This makes it painless to switch between stream source & sink implementations as your application requirements change.

Flink provides the [Kinesis Streaming Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/connectors/kinesis.html) to incorporate Kinesis into Flink application pipelines. The connector **is not published** in any public Maven repositories due to licensing concerns surrounding the AWS Java SDK. This means that users of the Kinesis Streaming Connector **must build the connector source on their own** and bundle the resulting compiled class files with their application JAR or with their Flink server installation.

This might seem daunting, but I assure you it's not that bad.

# Building the Kinesis Connector
Prerequisites for building Flink are:
 * Java JDK 6 or higher (Java 8 recommended)
 * Maven 3.x

We'll begin by downloading the source for Apache Flink. The latest stable release at this time is 1.2.0, so that's what I 'll use:
{% gist skidder/726e8abd3428833bff87862655754e5c %}


Enter the source directory, change the Scala version to 2.11 and build the core Flink components. This will take some time (typically 5-10 minutes), so I suggest getting a fresh cup of coffee while it builds.
{% gist skidder/384c6d05df50a3be6e1a13cafd902ecb %}


Lastly, change directories to the Kinesis streaming connector source. You can build this as-is; however, I prefer to update the AWS SDK versions to something more recent. These versions can be overriden using properties given in the Maven command:
{% gist skidder/dac095fc83b75d3d07897f7cdcbc29ef %}


# Using the Kinesis Connector with your Application
Install the Flink Kinesis streaming connector in your local Maven repository so that it can be included in your application's Maven-managed dependencies. I recommended excluding the Kinesis connector from your application's shade JAR since the AWS dependencies amount to several Megabytes and bloat the size of your application JAR. Instead, place the Kinesis connector JAR in the `lib` sub-directory of the Flink installation. This will lead to the Kinesis connector and its dependencies being available to all Flink applications running in the cluster.

# Configuration
As is common for services deployed with Docker, a lot of my configuration is specified through environment variables. The following Java snippet shows how I read environment-variables into a Java `Properties` object that can be supplied to the Kinesis connector:
{% gist skidder/868d6ffaebe7cbeee708cec292336a4f %}


My services communicate using Protobufs, so I've created a custom deserialization type to give access to the raw bytes:
{% gist skidder/27bc678f28d008d13443375b01f08423 %}


Lastly, instantiate the Kinesis consumer:
{% gist skidder/7f7ccc87f40ea6e1e84d9b5c74f39db2 %}


# Conclusion
I hope this inspires you to try the Flink Kinesis streaming connector with your own application. It's a small amount of effort to get things set up, but well worth it in the end!

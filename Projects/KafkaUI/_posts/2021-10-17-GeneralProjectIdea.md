---
layout: post
title: KafkaUI - General Idea
author: James Clancy
tags: fsharp dotnet
---

## Overview
I recently attended the Kafka Summit Americas and have been experimenting with Kafka. One thing I have found lacking with Kafka vs other Pub-Sub systems is a management UI. I have previously used RabbitMQ and Azure Service Bus and both of those had quite useful tooling. Unfortunately, Kakfa largely utilizes helpful but not user friendly command line interfaces or commercial products. There appear to be a few user friendly clients like [KaDrop](https://github.com/obsidiandynamics/kafdrop) and [Conduktor](https://www.conduktor.io/) but these are still quite limited vs what you get built into something like Rabbit. 

My thought was that it might be a useful try to build a new basic admin interface for Kafka, not one that would actually compete with the existing solution but rather one which fit what I needed and provide a good way to get more comfortable with Kafka. Overall, I don't think this is all that viable *but* it did create a great environment to come up with some really informative if hacky solutions to short comings in the dotnet `AdminClient` in `Confluent.Kafka`.

## The Existing tooling for dotnet - `Confluent.Kafka`

The core client for interfacing with Kafka in the dotnet ecosystem is the `Confluent.Kafka` library, this appears to be quite limited vs the Java client library. Some limitations appear to be:

1. Legacy style APIs which don't all support async/await
2. No streaming support
3. Very limited administrative capabilities

These appear to be known limitation related to the underlying [librdkafka](https://github.com/edenhill/librdkafka). Generally, the client allows more general use cases of consuming and producing events but more advanced functionality.

Reading online, it appears that the guidance is not to try to manage the cluster directly using librdkafka client implementations but rather utilize the [Confluent REST APIs](https://docs.confluent.io/platform/current/kafka-rest/index.html) which does look great but unfortunately is a Confluent (i.e. commercial) product and not a real part of Kafka itself, for now it seems like *good* solutions require Java or to utilize Confluent Licensed tools. 

Out of general interest I wanted to see if I could come up with hacky solutions to try to perform similar functionality to Conduktor using only `Confluent.Kafka` and interfacing with `Zookeeper`.

### Getting the Topic Offsets & Event Counts

Even looking through the [Rest Proxy Api Specs](https://docs.confluent.io/2.0.0/kafka-rest/docs/api.html#consumers) I am not seeing this info available in the rest api. 

Basically, for each topic I believe I want to compare the 
`kafka-run-class kafka.tools.GetOffsetShell --broker-list=localhost:29092 --topic=testtopic --time -1` results to the `kafka-run-class kafka.tools.GetOffsetShell --broker-list=localhost:29092 --topic=testtopic --time -2` for topics.

![Get Offset Example](\assets\img\post-media\2021-10-17-GeneralProjectIdea\get-offset-shell-example.png)

For consumer groups I believe we want to simulate `kafka-consumer-groups --bootstrap-server=localhost:29092 --group=testgroup --describe`.

![Consumer Groups Example](\assets\img\post-media\2021-10-17-GeneralProjectIdea\kafka-consumer-groups-example.png)


## Links

* [Visit the Repository](https://github.com/jamesclancy/KafkaUI)
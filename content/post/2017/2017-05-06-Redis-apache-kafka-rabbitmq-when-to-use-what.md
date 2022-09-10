---
date: 2017-05-06
dateName: '06 May 2017'
title: 'Redis, Apache Kafka & RabbitMQ - When to use what'
slug: redis-kafka-rabbitmq
tags:
  - Open source
---

I recently had to present a design where the designed consisted of [Redis](https://redis.io/), Apache [Kafka](https://kafka.apache.org/) and [RabbitMQ](https://www.rabbitmq.com/), among other things of course. At the presentation the obvious question came up, can't we just use one of thesse?

I understand that for novice users visiting the respective websites it must be difficult to understand the differences. On the front page on the redis webpage they state 

> Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker.

On the Apache Kafka webpage you'll see

> Publish & Subscribe to streams of data like a messaging system

and finally on RabbitMQ

> RabbitMQ is the most widely deployed open source message broker 

"Message" seems to be the keyword for all of them but that doesn't tell the full story. Let's have a look at the details and example scenarios where one would choose one over the other.

====

# Redis

The main thing that redis does is being a blazing fast in-memory data structure store and as such it's mainly seen in caching related scenarios. After release 3 however, many features were added and redis now is also wildly used as a publish/subscribe messaging system where it crosses into RabbitMQ territory.

On this site I use redis for caching the responses. If a user visits a newly created URL that nobody has visited before it will be served by the DB but the HTML response will be cached in redis. Next time somebody visits the same URL the request is served from cache without hitting the DB.

As mentioned before, redis is also widley used as a publish/subscribe messaging system. I have seen it being utilized in simple chat applications as well as a message broker for distributed tasks queues.

# Apache Kafka

The main thing kafka does is provide you with the ability to perform real-time processing of streaming data. What is streaming data? Syslog messages send to you, stock price updates send to you, any data clients might generate that your servers receive can be thought of as streaming data.

Like redis, kafka is also blazing fast but it is limited to the I/O of the hard drive however, persistence is higher with Kafka. Kafka can store data on multiple machines in a fault tolerant manner and supports automatic recovery when machine failures occur.

Kafka design is build around the producer-consumer model. Producers write to topics, many topics can be written to a partition and consumers consume these topics. Queuing and publish/subscribe are traditionaly the main two models in messaging and Kafka is flexible enough to incorperate both of them.

# RabbitMQ

RabbitMQ provides a fault tolerant messaging queue. It can be used to build publish/subscribe and work queuing solutions. RabbitMQ supports message acknowledgments. An acknowledgement is sent back from the consumer to tell RabbitMQ that a particular message had been received, processed and that RabbitMQ is free to delete it.

If a consumer dies (its channel is closed, connection is closed, or TCP connection is lost) without sending an ACK, RabbitMQ will understand that a message wasn't processed fully and will re-queue it. If there are other consumers online at the same time, it will then quickly redeliver it to another consumer. That way you can be sure that no message is lost, even if the workers occasionally die.

# Redis vs RabbitMQ

Redis main application is in memory storage. For pub/sub related applications I would prefer RabbitMQ over Redis as you get persistence, at least once delivery guarantees and complex topic based routing features out of the box.

# Redis vs Kafka

I can see a use case where one could use redis in memory storage for ingestion of streaming short lived data with high consumer capacity on the backend. If you go this route make sure to keep an eye on memory usage. If consumer capacity is a limit I would advice to use kafka since kafka is built to accept and persist vast sums of data regardless of any consumers.

# RabbitMQ vs Kafka

The same applies here as with redis. RabbitMQ was not designed for streaming message ingestion. If you want to use it as such make sure to have enough consumer capacity on the backend and preferably really fast ones. If the message queue grows to large RabbitMQ will stop responding which will lead to problems. Kafka on the other hand was specifically designed for this purpose. Kafka also makes it easy for multiple consumers to consume the same topic. If you encountered problems later on in your application you can playback a topic and let the consumer do its work again. With RabbitMQ after it receives an ACK the message is deleted and will not be seen again in the queue.

<br />
<br />

# Summary table

|    | Redis | Kafka | RabbitMQ |
| -- | ----- | ----- | -------- |
| Streaming data |    |  X  |     |
| ETL |    |  X  |     |
| Message routing |    |     |  X  |
| Topics |    |  X  |  X   |
| HA Storage |    |  X  |  X   |
| Chat application |  X  |     |  X  |
| High throughput |  X  |  X  |     |



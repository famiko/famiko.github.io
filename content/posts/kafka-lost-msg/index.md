---
title: '怎么保证kafka不丢失消息'
date: 2023-12-17T17:33:59+08:00
description: '怎么保证kafka不丢失消息，消费者Consumer如何提交offset，生产者Producer如何配置acks，broker副本ISR介绍'
tags: [kafka]
---
# 前言
整理自己kafka的相关知识，选了几个点展开。
# 正文
## Consumer
Consumer如果对topic从头开始消费，在不考虑broker上分区过期log被删除的情况下，就不会出现消息丢失的情况了。这种情况只需考虑重复消费和追赶消费进度的问题。对于消费进度的问题，可以使用offset解决。  
通过offset机制，Consumer可以从上次提交的offset继续消费消息。Consumer可以通过配置进行定时自动提交offset，但是当提交offset时应用还没有处理完这批消息挂掉或者重启的话，新的消费者会从已提交的offset继续消费，之前的消息就丢失了没有处理。  
对于无法容忍消息丢失的场景，可以使用手动提交offset的方式，即应用处理完这批消息再提交offset来避免。
根据Consumer客户端提供的API，可以使用同步提交或者异步提交的方式。

以下代码是一个单线程消费和同步提交的代码，考虑吞吐量，需要注意循环块内部代码的执行性能和offset同步提交对线程的阻塞。
```java
while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records) {
        // process record
        System.out.println(record);
     } 
     consumer.commitSync();
}
```
存在消费线程和处理线程等多线程处理的情况下，可以考虑使用`commitSync(Map<TopicPartition,OffsetAndMetadata> offsets)
`方法或者对应的异步方法，来提交处理完的消费位置。

对于重复消费，应用可以考虑去重或者幂等的方式来处理。

## Producer
Producer主要考虑内存中的发送buffer断电丢失和acks的配置。

对于buffer断电丢失的问题，比较常见的是使用本地消息表的模式，记录待发送的消息和已发送的位置信息。

对acks配置，当消息发送到Kafka集群时，Producer可以要求Kafka给予不同级别的确认。
* acks=0：生产者发送消息后不需要等待确认，存在丢失消息的风险，因为Kafka并不会确认消息是否被成功写入。
* acks=1：生产者发送消息后会等待分区Leader确认消息写入成功后才会收到确认。
* acks=all（或acks=-1）：生产者发送消息后等待ISR（In-Sync Replicas，同步副本集）中所有副本都写入成功后才会收到确认。

一般来讲，broker为了追求吞吐量，不会设置成对每条消息进行刷盘，那么为了消息不丢失，可以考虑使用副本以及将Producer配置成`acks=all`。
## Broker
如前所述，broker往往不会同步刷盘，而是配置副本来解决持久化的的问题。比较关键的配置参数是`min.insync.replicas`，作用是当分区ISR大于等于配置参数的个数时才允许写入。
举个例子，topic一个分区有5个副本，`min.insync.replicas=3`，则允许2个副本故障。
# 最后
简单地从几个方面泛泛而谈，后续可能会有其它相关的文章。
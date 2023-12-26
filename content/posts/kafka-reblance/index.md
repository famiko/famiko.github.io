---
title: '对于kafka的重平衡机制需要了解什么'
date: 2023-12-21T22:06:52+08:00
description:
tags:
featured_image:
featured_image_banner_process: ''
featured_image_thumbnail_process: ''
draft: false
---
# 前言
对于单个Consumer而言，它可以订阅消费一些topic下所有分区的消息。考虑吞吐量的话，单个Consumer是存在性能上限的；考虑可用性的话，如果单个Consumer挂了或者跟broker之间网络出现问题，消费就暂停了。所以非常自然的想法是有一组Consumer携手处理这些topic各个分区的数据。

kafka提供了消费者组（consumer group）的概念将这些Consumer关联在一起，Consumer和分区是一对多的关系，每个分区分配给最多一个Consumer，每个Consumer可以消费多个分区。而重平衡机制指的是当消费者加入或退出消费者组时，这些Consumer是怎么协调的。另外当topic分区变化时也会导致重平衡。

![](./1343650260-8ec8ea068cc59ceb_fix732.webp#center)

本文会以QA的方式展开
# 正文 
## Q1:怎么判断Consumer加入和退出group，是有相关心跳的机制吗？
是的。Consumer和broker通过心跳保证存活性。Consumer可以配置心跳间隔（`heartbeat.interval.ms`）和会话超时时间（`session.timeout.ms`）。当会话超时没有收到心跳时，则判断Consumer退出group。

除了上面两个参数，还需要关注Consumer的最大拉取间隔（`max.poll.interval.ms`）。如果应用没有及时调用`poll()`方法，也会触发重平衡。这种情况一般发生在消费线程无法在间隔时间里处理完数据或者消费线程阻塞以及挂掉的情况。

## Q2:Consumer报心跳给哪个broker，怎么把分区分配给Consumer？
这里有一个组协调者的角色（group coordinator），每个group都有一个，Consumer将心跳报给组协调者。Consumer初始通过`bootstrap.servers`连接到kafka集群，再拿到集群的拓扑信息和元数据，就可以知道哪个broker是组协调者，再与它保持心跳。可以参考下图

![](./consumer-first-poll.png#center)

分区的话是通过JoinGroup和SyncGroup请求获取的。下面是JoinGroup的结果处理方法，可以看到有Leader和Follower两种角色处理。
```java
// kafka 2.8 省略部分代码
...
    private class JoinGroupResponseHandler extends CoordinatorResponseHandler<JoinGroupResponse, ByteBuffer> {
        private JoinGroupResponseHandler(final Generation generation) {
            super(generation);
        }

        @Override
        public void handle(JoinGroupResponse joinResponse, RequestFuture<ByteBuffer> future) {
            Errors error = joinResponse.error();
            if (error == Errors.NONE) {
                ...
                            if (joinResponse.isLeader()) {
                                onJoinLeader(joinResponse).chain(future);
                            } else {
                                onJoinFollower().chain(future);
                            }
...
```
看下Leader部分是怎么处理的。主要是先将分区分配给group内各个消费者，再发送SyncGroup请求上报分区结果。
```java
    private RequestFuture<ByteBuffer> onJoinLeader(JoinGroupResponse joinResponse) {
        try {
            // perform the leader synchronization and send back the assignment for the group
            Map<String, ByteBuffer> groupAssignment = performAssignment(joinResponse.data().leader(), joinResponse.data().protocolName(),
                    joinResponse.data().members());

            List<SyncGroupRequestData.SyncGroupRequestAssignment> groupAssignmentList = new ArrayList<>();
            for (Map.Entry<String, ByteBuffer> assignment : groupAssignment.entrySet()) {
                groupAssignmentList.add(new SyncGroupRequestData.SyncGroupRequestAssignment()
                        .setMemberId(assignment.getKey())
                        .setAssignment(Utils.toArray(assignment.getValue()))
                );
            }

            SyncGroupRequest.Builder requestBuilder =
                    new SyncGroupRequest.Builder(
                            new SyncGroupRequestData()
                                    .setGroupId(rebalanceConfig.groupId)
                                    .setMemberId(generation.memberId)
                                    .setProtocolType(protocolType())
                                    .setProtocolName(generation.protocolName)
                                    .setGroupInstanceId(this.rebalanceConfig.groupInstanceId.orElse(null))
                                    .setGenerationId(generation.generationId)
                                    .setAssignments(groupAssignmentList)
                    );
            log.debug("Sending leader SyncGroup to coordinator {} at generation {}: {}", this.coordinator, this.generation, requestBuilder);
            return sendSyncGroupRequest(requestBuilder);
        } catch (RuntimeException e) {
            return RequestFuture.failure(e);
        }
    }
```
看下Follower是怎么处理的。主要是发送SyncGroup请求拿到分配的分区。
```java
    private RequestFuture<ByteBuffer> onJoinFollower() {
        // send follower's sync group with an empty assignment
        SyncGroupRequest.Builder requestBuilder =
                new SyncGroupRequest.Builder(
                        new SyncGroupRequestData()
                                .setGroupId(rebalanceConfig.groupId)
                                .setMemberId(generation.memberId)
                                .setProtocolType(protocolType())
                                .setProtocolName(generation.protocolName)
                                .setGroupInstanceId(this.rebalanceConfig.groupInstanceId.orElse(null))
                                .setGenerationId(generation.generationId)
                                .setAssignments(Collections.emptyList())
                );
        log.debug("Sending follower SyncGroup to coordinator {} at generation {}: {}", this.coordinator, this.generation, requestBuilder);
        return sendSyncGroupRequest(requestBuilder);
    }

```
## Q3:哪个Consumer会成为group中的Leader角色？
如果这个group没有leader的话，那么第一个被处理JoinGroup请求的Consumer会成为leader。如下面代码所示，使用了锁来处理相关逻辑。
```
  ...
  private def doJoinGroup(group: GroupMetadata,
                          memberId: String,
                          groupInstanceId: Option[String],
                          clientId: String,
                          clientHost: String,
                          rebalanceTimeoutMs: Int,
                          sessionTimeoutMs: Int,
                          protocolType: String,
                          protocols: List[(String, Array[Byte])],
                          responseCallback: JoinCallback): Unit = {
    group.inLock {
        ...
```
## Q3:组协调器存在单点问题吗？
组协调器由内部的保存consumer group offset的topic的一个分区的leader担任。如果leader挂掉，剩下的follower会竞选新的leader。
## Q4:重平衡的流程是怎么样的？
可以参考下图。

加入一个新的group可以参考从Empty到PreparingRebalance、AwaitSync和Stable状态的变化。

现存group重平衡可以参考从Stable到PreparingRebalance、AwaitSync和Stable状态的变化。
![](./GroupStat.png#center)
# 参考资料
1. [https://github.com/apache/kafka/blob/2.8/clients/src/main/java/org/apache/kafka/clients/consumer/internals/AbstractCoordinator.java](https://github.com/apache/kafka/blob/2.8/clients/src/main/java/org/apache/kafka/clients/consumer/internals/AbstractCoordinator.java)
2. [https://kafka.apache.org/28/documentation.html](https://kafka.apache.org/28/documentation.html)
3. [https://chrzaszcz.dev/2019/06/16/kafka-consumer-poll/](https://chrzaszcz.dev/2019/06/16/kafka-consumer-poll/)
4. [https://matt33.com/2017/10/22/consumer-join-group/#consumer-offsets-topic](https://matt33.com/2017/10/22/consumer-join-group/#consumer-offsets-topic)
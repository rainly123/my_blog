---
title: "kafka最佳实践"
date: 2021-12-28T15:39:00+08:00
description: PHP，golang工程师，项目管理，软件架构
draft: true
toc: true
---

## producer使用规范

1. acks (一般建议选择

   ```
   acks=1
   ```

   ，重要的服务可以设置

   ```
   acks=all
   ```

   ):

   1. `acks=0`：无需服务端的Response、性能较高、丢数据风险较大。
   2. `acks=1`：服务端主节点写成功即返回Response、性能中等、丢数据风险中等、主节点宕机可能导致数据丢失。
   3. `acks=all`：服务端主节点写成功且备节点同步成功才返回Response、性能较差、数据较为安全、主节点和备节点都宕机才会导致数据丢失。

2. batch(重要的服务可以设置

   linger.ms

   =0)

   1. `batch.size` : 发往每个分区（Partition）的消息缓存量（消息内容的字节数之和，不是条数）达到这个数值时，就会触发一次网络请求，然后客户端把消息真正发往服务器；
   2. `linger.ms` : 每条消息待在缓存中的最长时间。若超过这个时间，就会忽略`batch.size`的限制，然后客户端立即把消息发往服务器。
   3. `batch.size`有助于提高吞吐，`linger.ms`有助于控制延迟。您可以根据具体业务需求进行调整。

3. 重试

   1. retries，重试次数，建议设置为3。
   2. [retry.backoff.ms](http://retry.backoff.ms/)，重试间隔，建议设置为1000。

## consumer使用规范

1. 负载均衡

   1. cousumer数得小于等于partition数量，一条消息只能被同group下一个cousumer消费

2. 多个订阅

   1. 一个Consumer Group可以订阅多个Topic。
   2. 一个Topic也可以被多个Consumer Group订阅，且各个Consumer Group独立消费Topic下的所有消息。

3. 消费位点

   1. 每个Topic会有多个分区，每个分区会统计当前消息的总条数，这个称为最大位点MaxOffset。剩余的未消费的条数（也称为消息堆积量） = MaxOffset - ConsumerOffset
   2. 消息队列Kafka版 Consumer会按顺序依次消费分区内的每条消息，记录已经消费了的消息条数，称为ConsumerOffset。

4. 消费位点提交

   1. `enable.auto.commit`：默认值为true。
   2. `auto.commit.interval.ms`： 默认值为1000，也即1s
   3. 将`enable.auto.commit`设置为true，则需要在每次poll数据时，确保前一次poll出来的数据已经消费完毕，否则可能导致位点跳跃。
   4. 如果想自己控制位点提交，请把`enable.auto.commit`设为false，并调用`commit(offsets)`函数自行控制位点提交。
   5. 确保处理完消息后再做消息commit，避免业务消息处理失败，无法重新拉取处理失败的消息。

5. 消费位点重置

   1. ```
      auto.offset.reset
      ```

      来配置重置策略，主要有三种策略：

      - “latest”，从最大位点开始消费。
      - “earliest”，从最小位点开始消费。
      - “none”，不做任何操作，也即不重置。

   2. 第一次上线：建议设置成 “latest”，而不要设置成 “earliest”，避免因位点非法时从头开始消费，从而造成大量重复。

6. 消息重复和消费幂等

   1. 发送消息时，传入key作为唯一流水号ID；
   2. 消费消息时，判断key是否已经消费过，如果已经消费过了，则忽略，如果没消费过，则消费一次；

7. 消费失败

   1. 失败后一直尝试再次执行消费逻辑。这种方式有可能造成消费线程阻塞在当前消息，无法向前推进，造成消息堆积；
   2. 由于消息队列Kafka版没有处理失败消息的设计，实践中通常会打印失败的消息、或者存储到某个服务（例如创建一个Topic专门用来放失败的消息），然后定时check失败消息的情况，分析失败原因，根据情况处理。

8. 消费阻塞以及堆积

   1. 增加Consumer实例个数
   2. 增加消费协程

9. 消息过滤

   1. 如果过滤的种类不多，可以采取多个Topic的方式达到过滤的目的；
   2. 如果过滤的种类不多，可以采取多个Topic的方式达到过滤的目的；

10. 补充

    1. consumer数量不能超过topic分区数，否则会有consumer拉取不到消息。
    2. cnsumer不能频繁加入和退出group，频繁加入和退出，会导致consumer频繁做rebalance，阻塞消费
    3. consumer需周期poll，维持和server的心跳，避免心跳超时，导致consumer频繁加入和退出，阻塞消费。
    4. consumer session设置为30秒，[session.timeout.ms](http://session.timeout.ms/)=30000。
    5. 消费线程退出要调用consumer的close方法，避免同一个组的其他消费者阻塞[sesstion.timeout.ms](http://sesstion.timeout.ms/)的时间。

**topic使用规范:**

1. 配置要求：推荐3副本，同步复制，最小同步副本数为2，且同步副本数不能等于topic副本数，否则宕机1个副本会导致无法生产消息。

   创建方式：支持选择是否开启kafka自动创建Topic的开关。选择开启后，表示生产或消费一个未创建的Topic时，会自动创建一个包含3个分区和3个副本的Topic。

   单topic最大分区数建议为20。

包使用：

1. go
   1. https://github.com/Shopify/sarama
   2. https://code.shihuo.cn/go/hms-components/tree/master/mq/kafka
2. php
   1. https://code.shihuo.cn/php-composer/kafka
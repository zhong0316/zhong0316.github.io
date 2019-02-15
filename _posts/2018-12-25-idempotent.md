---
layout: post
title: MQ消费端的幂等
categories: [Java, MQ]
description: MQ消费端的幂等
keywords: MQ, 幂等
---

# MQ消费端的幂等
MQ消费端在接收到MQ消息之后按照业务key(uuid)进行防重，达到消费的幂等性。

## 业务场景
用户在使用白条优惠劵打白条支付订单后，如果用户整单退款，需要给用户补发优惠劵。白条异步处理系统监听白条退款MQ消息进行整单退退劵操作。对于一笔订单，整单退之后，用户的优惠劵只能补发一次。因此需要对MQ消费做防重幂等操作。
### 消费幂等的必要性
因为网络会发生抖动，MQ消费端和生产端都可能会出现超时，这样MQ就会出现重复发送和重复消费的情况，因此需要做到消费幂等。防止因此可能产生的资产损失。
### 整单退退劵的幂等  
异步处理系统在接收到退款消息之后，会判断当前订单是否已经整单退款，如果已经整单退款，首先会恢复用户的参与次数（库存），然后会写退款流水记录(refund_record)。这两个步骤的操作是一个事务。退款流水记录的'uuid'字段用来防重（加唯一索引），'uuid'的生成规则为'userId' + 'orderId'。这样在MQ重试或者并发时spring会抛出[DuplicateKeyException](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/dao/DuplicateKeyException.html)。我们捕获这个异常，如果是这个异常，默认MQ消费成功。这样就可以做到MQ消费的幂等性。简单地说，就是通过数据库的**唯一索引**来保证MQ消费的幂等。
  流程图如下：
  
<div align="center">
    <img src="{{ site.url }}/images/posts/biz/wholerefund.jpg" alt="整单退流程图"/>
</div>

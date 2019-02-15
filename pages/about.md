---
layout: page
title: 关于我
description: cafesolo
keywords: 赵忠伟
comments: true
menu: 关于
permalink: /about/
---

我是忠伟，一个喜欢篮球和黑咖啡的程序员。

## 教育背景

| 时间 | 学校 | 专业 | 学历 |
| :------: | :------: | :------: | :------: |
| 2014.9-2017.6 | 武汉大学 | 管理科学与工程 | 研究生 |
| 2009.9-2013.6 | 武汉大学 | 信息管理与信息系统 | 本科 |

##  工作经历
**时间**：2017.07-⾄今  
**公司**：京东⾦融  
**职位**：软件开发工程师  
**工作内容**：
1. ⽩白条内单营销:⽩白条营销活动按照优惠类型分为:本⾦金金类优惠(满减、⽴立减、随机⽴立减)活动和
 息费折扣类活动。参与⽩白条营销活动的整个⽣生命周期开发，包括:⽩白条营销活动创建、活动上下
 架、优惠劵领取、活动匹配、选取优惠劵、优惠劵消费、⽤用户退款退还优惠劵等。
2. ⽩白条外单(闪付)营销:⽩白条外单主要针对线下⽩白条闪付⽀支付场景。主要负责活动管理理平台、活
 动匹配、⽤用户退款后退还优惠劵、商户结算。

## 个人技能
* 熟悉Java多线程并发，锁。了了解JVM，Java内存模型。
* 熟悉基本的数据结构和算法，熟悉Java集合框架。Leetcode 490+。
* 熟悉IO、NIO。了了解NIO在⾼高性能服务端的⼴广泛应⽤用，了了解分布式治理理框架。阅读过Dubbo、
Netty等的源码。了了解Dubbo、Netty的基本架构。了了解服务端线程池模型演进过程。
* 熟悉MySQL，了了解数据库基本理理论和常⽤用优化技术，了了解分布式缓存，了了解Redis。了了解
Zookeeper。
* 了了解分布式系统基本理理论。了了解服务治理理框架、消息队列列。了了解前后端分离技术。
* 了了解Flink，对实时计算有⼀一定的认识。
* 有搜索引擎和⽹网络爬⾍虫经验。

## Github
{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}

---
layout: page
title: 关于我
description: 关于我
keywords: 赵忠伟
comments: true
menu: 关于
permalink: /about/
---

我是赵忠伟，一个喜欢篮球的程序员。

## 教育背景

| 时间 | 学校 | 专业 | 学历 |
| :------: | :------: | :------: | :------: |
| 2014.9-2017.6 | 武汉大学 | 管理科学与工程 | 研究生 |
| 2009.9-2013.6 | 武汉大学 | 信息管理与信息系统 | 本科 |

## 个人技能
* 熟悉Java多线程并发，锁。了解JVM，Java内存模型。
* 熟悉基本的数据结构和算法，熟悉Java集合框架。Leetcode 490+。
* 熟悉IO、NIO。了解NIO在⾼性能服务端的⼴泛应用，了解分布式治理理框架。阅读过Dubbo、
Netty等的源码。了解Dubbo、Netty的基本架构。了解服务端线程池模型演进过程。
* 熟悉MySQL，了解数据库基本理论和常⽤优化技术，了解分布式缓存，了解Redis。了解
Zookeeper。
* 了解分布式系统基本理理论。了解服务治理理框架、消息队列。了解前后端分离技术。
* 了解Flink，对实时计算有⼀定的认识。
* 有搜索引擎和网络爬虫经验。

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

---
layout: post
title: 网际控制报文协议ICMP
data: 2017/5/17
subtitle: ""
description: ""
thumbnail: "/img/计网/ICMP.jpeg"
tags:
  - 计算机网络#网络层
categories:
  - 计算机网络
---

# 网际控制报文协议ICMP

> ## 目录

* 概述
* ICMP报文种类
* ICMP应用举例

> ### 概述

为了更有效地转发IP数据报和提高交付成功的机会, 在网际层使用了网际控制报文协议ICMP(Internet Control Message Protocol). ICMP允许主机或者路由器报告差错情况和提供有关异常情况的报告.ICMP报文作为IP层的数据报的数据.

> ### ICMP报文种类

* 网际报文的种类有两种, 即ICMP差错报告报文和ICMP询问报文.
* 几种常用的ICMP报文类型

| ICMP报文类型 | 类型的值 | ICMP报文的类型 |
| ------ | ------ | ------- |
| 差错报告报文 | 3 | 终点不可达 |
| 差错报告报文 | 4 | 源点抑制 |
| 差错报告报文 | 11 | 时间超过 |
| 差错报告报文 | 12 | 参数问题 |
| 差错报告报文 | 5 | 改变路由 |
| 询问报文 | 8或0 | 回送请求或回答 |
| 询问报文 | 13或14 | 时间戳请求或回答 |

#### 终点不可达

当路由器或者主机不能交付数据报时就向源点发送终点不可达报文

#### 源点抑制

当路由器或者主机因拥塞而丢弃数据报时, 就向源点发送源点抑制报文, 使源点知道应当把数据报的发送速率放慢.

#### 时间超过

当路由器收到生存时间为0的数据报时, 除了丢弃数据报外, 还要向源点发送时间超过报文.

#### 参数问题

当路由器或者目的主机收到的数据报的首部中有的字段的值不正确时, 就丢弃该数据报,并向源点发送参数问题报文.

#### 改变路由(路由重定向)

路由器把改变路由报文发送给主机, 让主机知道下次应该将数据报发送给另外的路由器.

#### 回送请求和回答

ICMP回送请求报文是由主机或者路由器向一个特定的目的主机发送询问. 收到此报文的主机必须给源主机或者路由发送ICMP回送回答报文.

#### 时间戳请求和回答

ICMP时间戳请求报文是请某个主机或者路由器回答当前的日期和时间.

> ### ICMP的应用举例

#### PING

* ICMP一个重要应用就是分组网间探测PING, 用来测试两个主机之间的连通性.
* PING使用了ICMP的回送请求和回送回答报文.
* PING是应用层直接使用网络层ICMP的例子, 它没有通过运输层的TCP或UDP.

#### traceroute

* traceroute是用来跟踪一个分组从源点到终点的路径.
* traceroute从源主机向目的主机发送一连串IP数据报, 数据报中封装的是无法交付的UDP用户数据报.第一个数据报P1的生存时间为TTL设置为1.当P1到达路径上的第一个路由器R1时, 路由器R1先收下它, 接着把TTL的值减1.由于TTL等于0了, R1就把P1丢弃了, 并向源主机发送一个ICMP时间超过的差错报告报文.
* 如此循环下去, 当最后一个数据报刚刚达到目的主机时, 数据报的TTL是1, 主机不转发数据, 也不把TTL减1.但因IP数据报封装的是无法交付的运输层UDP用户数据报, 因此目的主机要向源主机发送ICMP终点不可达差错报告报文.

> ## 参考资料

> * 计算机网络(谢希仁)

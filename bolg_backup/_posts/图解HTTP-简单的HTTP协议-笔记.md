---
title: 图解HTTP-简单的HTTP协议-笔记
data: 2017/2/17
tags:
  - 计算机网络#HTTP
categories:
  - 计算机网络
---

#### 通过请求和响应来交换信息

![](/uploads/图解HTTP/简单的HTTP协议01.jpg)
<!-- more -->
* 请求报文是由请求方法、请求 URI、协议版本、可选的请求首部字段和内容实体构成的。
* 请求报文的构成

![](/uploads/图解HTTP/简单的HTTP协议02.jpg)

* 响应报文基本上由协议版本、状态码(表示请求成功或失败的数字代码)、用以解释状态码的原因短语、可选的响应首部字段以及实体主体构成。

![](/uploads/图解HTTP/简单的HTTP协议03.jpg)

#### HTTP是不保存状态的协议
* HTTP协议自身不对请求和响应之间的状态进行保存, 也就是HTTP不对请求和响应做持久化处理
* 不保存状态这种设计的优点:更快的处理大量事物

#### 告知服务器意图的HTTP方法

* GET:获取资源

![](/uploads/图解HTTP/简单的HTTP协议04.png)

* POST:传输实体主体
* PUT:PUT 方法用来传输文件。就像 FTP 协议的文件上传一样,要求在请求报文的主体中包含文件内容,然后保存到请求 URI 指定的位置。36但是,鉴于 HTTP/1.1 的 PUT 方法自身不带验证机制,任何人都可以上传文件 , 存在安全性问题,因此一般的 Web 网站不使用该方法。若配合 Web 应用程序的验证机制,或架构设计采用REST(REpresentational State Transfer,表征状态转移)标准的同类Web 网站,就可能会开放使用 PUT 方法。
* HEAD:获得报文首部, 返回的不包含主体部分,常用于URI的有效期以及更新日期
* OPTIONS:询问支持的方法, OPTIONS 方法用来查询针对请求 URI 指定的资源支持的方法。
* CONNECT:要求用隧道协议连接代理。CONNECT 方法要求在与代理服务器通信时建立隧道,实现用隧道协议进行 TCP通信。主要使用 SSL(Secure Sockets Layer,安全套接层)和 TLS(Transport Layer Security,传输层安全)协议把通信内容加 密后经网络隧道传输。

![](/uploads/图解HTTP/简单的HTTP协议05.jpg)

#### 持久连接减少通信量

在HTTP协议初始的几个版本中, 每次HTTP请求都得断开一次, 这样TCP频繁的连接和断开会增加通信量的开销

![](/uploads/图解HTTP/简单的HTTP协议06.jpg)

* 为了解决该问题, 在HTTP1.1和部分HTTP1.0中想出了持久连接(HTTP Persistent Connections,也称为 HTTP keep-alive 或
HTTP connection reuse)的方法。该方法的特点是:只要任意一端没有明确提出断开连接,则保持 TCP 连接状态。

![](/uploads/图解HTTP/简单的HTTP协议07.jpg)

* 线管化技术
在之前的HTTP请求都是要一个请求完另外一个才能继续,但是持久化连接使得请求线管化成为了可能。即一次性可以请求多个

![](/uploads/图解HTTP/简单的HTTP协议08.jpg)

#### 使用Cookie的状态管理

* HTTP是无状态协议,无状态指的是它不对之前发生的请求和响应进行管理。例如:一个需要登录的Web网站,无法对是否登录进行状态管理,每次跳转页面不是要重新登录就是要在报文中添加附加的数据来进行登录状态管理
* 无协议状态的优点:减少服务器的CPU及内存消耗
* 为了解决这一矛盾,由此引进了Cookie技术来进行状态管理

##### Cookie技术简介

* Cookie 会根据从服务器端发送的响应报文内的一个叫做 Set-Cookie 的首部字段信息,通知客户端保存 Cookie。当下次客户端再往该服务器发送请求时,客户端会自动在请求报文中加入 Cookie 值后发送出去。

* 服务器端发现客户端发送过来的Cookie后,会去检查究竟是从哪一个客户端发来的连接请求,然后对比服务器上的记录,最后得到之前的状态信息。
* 没有Cookie下的请求:

![](/uploads/图解HTTP/简单的HTTP协议09.jpg)

* 第 2 次以后(存有 Cookie 信息状态)的请求

![](/uploads/图解HTTP/简单的HTTP协议10.jpg)

> 以上笔记来源于图解HTTP一书

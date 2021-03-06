---
title: 图解HTTP-返回结果的状态码-笔记
data: 2017/2/17
tags:
  - 计算机网络#HTTP
categories:
  - 计算机网络
---

#### 状态码类别：

类别	原因短语
1XX	Informational(信息性状态码)	接收的请求正在处理
2XX	Success(成功状态码)	请求正常处理完毕
3XX	Redirection(重定向状态码)	需要进行附加操作以完成请求
4XX	Client Error(客户端错误状态码)	服务器无法处理请求
5XX	Server Error(服务器错误状态码)	服务器处理请求出错

<!-- more -->

##### 2XX 成功

* 200 OK：表示客户端的请求被服务器成功处理
* 204 No Content：该状态码代表服务器接收的请求已成功处理,但在返回的响应报文中不含实体的主体部分。另外,也不允许返回任何实体的主体。比如,当从浏览器发出请求处理后,返回 204 响应,那么浏览器显示的页面不发生更新。

* 206 Partial Content：该状态码表示客户端进行了范围请求,而服务器成功执行了这部分的GET请求。响应报文中包含由Content-Range 指定范围的实体内容。

##### 3XX 重定向

3XX 响应结果表明浏览器需要执行某些特殊的处理以正确处理请求。

* 301 Moved Permanently:永久性重定向。该状态码表示请求的资源已被分配了新的 URI,以后应使用资源现在所指的 URI。也就是说,如果已经把资源对应的 URI保存为书签了,这时应该按 Location 首部字段提示的 URI 重新保存。

* 302 Found:临时性重定向：该状态码表示请求的资源已被分配了新的URI，希望用户本次请求能使用新的URI。302和301很相似，302表示的URI并不是永久性移动，换句话说就是URI还有可能发生变化。

* 303 See Other：该状态码表示由于请求对应的资源存在着另一个 URI,应使用 GET方法定向获取请求的资源。303和302有着相似的功能，他们之间的区别是303状态码表示明确客户端采用GET请求获取资源

* 304 Not Modified  

##### 4XX客户端错误

4XX响应结果表示错误发生在客户端

* 400 Bad Request:该状态码表示请求报文中存在语法错误。当错误发生时,需修改请求的内容后再次发送请求。另外,浏览器会像 200 OK 一样对待该状态码。

* 401 Unauthorized：该状态码表示发送的请求需要有通过 HTTP 认证(BASIC 认证、DIGEST 认证)的认证信息。

* 403 Forbidden：该状态码表明对请求资源的访问被服务器拒绝了。

* 404 Not Found：该状态码表明服务器上无法找到请求的资源。

##### 5XX服务器错误

5XX 的响应结果表明服务器本身发生错误。

* 500 Internal Server Error:该状态码表明服务器端在执行请求时发生了错误。

* 503 Service Unavailable：该状态码表明服务器暂时处于超负载或正在进行停机维护,现在无法处理请求。

> 以上笔记来源于图解HTTP一书

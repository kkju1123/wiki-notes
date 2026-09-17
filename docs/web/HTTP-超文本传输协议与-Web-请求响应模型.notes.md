# HTTP：超文本传输协议与 Web 请求响应模型

*原文: [https://en.wikipedia.org/wiki/HTTP](https://en.wikipedia.org/wiki/HTTP) · 来源: web · 生成时间: 2026-09-17T03:56:59.911646+00:00*

## 背景

HTTP 诞生于 1989-1991 年 CERN 的科研信息共享需求，最初目的是让分散的文档可以通过超链接互相引用并跨网络获取。在 HTTP 之前，FTP/Gopher 等协议偏重文件下载或目录浏览，缺乏对超媒体和统一资源定位的简化抽象。HTTP 将“获取资源”统一为基于 URL 的方法调用，使浏览器、服务器、代理和缓存可以协同工作，成为 Web 架构的粘合剂。

## 痛点

如果不理解 HTTP，调试接口时看到 401/403/502 等状态码会无从下手，也不清楚 Cache-Control、Host、Content-Type 等头部如何影响行为。做前端、后端或运维时，无法定位跨域、缓存、连接复用等性能与故障问题。面试中 HTTP 是必考网络基础，只背状态码而不懂报文和版本演进很难深入回答。

## 解决办法

HTTP 的核心是客户端-服务器请求-响应模型：客户端用请求行（方法、URL、版本）说明“我要做什么”，服务器用状态行（版本、状态码、原因短语）说明“结果如何”，双方通过头部交换元数据，正文承载资源表示。它默认基于 TCP 提供可靠字节流，协议本身无状态，每个请求独立，因此服务端横向扩展相对简单。HTTP/1.1 通过 Connection: keep-alive 持久连接复用 TCP，减少三次握手和慢启动开销；Cache-Control/ETag 等头部进一步允许中间节点缓存，减轻服务器压力。可以把 HTTP 类比为餐厅点餐：顾客递单子（请求），后厨出菜并告知成功或失败（响应），服务员和传菜口相当于代理/缓存，单子本身不保存客人历史（无状态）。

## 关键代码示例

```http
GET /api/users/42 HTTP/1.1
Host: www.example.com
Accept: application/json
User-Agent: Mozilla/5.0
Connection: keep-alive

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 23
Cache-Control: max-age=60

{"id":42,"name":"Alice"}
```

第一行是请求行，包含方法 GET、资源路径 /api/users/42 和协议版本 HTTP/1.1；Host 是 HTTP/1.1 强制头部，用于同一 IP 多域名虚拟主机路由；Connection: keep-alive 表示请求后可复用 TCP 连接。响应第一行是状态行，200 OK 表示资源获取成功；Content-Type 和 Content-Length 告诉客户端如何解析正文；空行后的 JSON 是真正返回的数据。

## 关键流程

1. 客户端解析 URL，提取 scheme、Host、端口和路径。
2. 通过 DNS 解析域名，建立 TCP 连接（HTTPS 先完成 TLS 握手）。
3. 客户端发送 HTTP 请求报文：请求行、头部、空行、可选正文。
4. 服务器根据方法、路径和头部执行业务逻辑，生成响应报文。
5. 客户端接收响应，根据状态码和头部进行渲染、缓存或跳转。
6. 按 Connection 头决定复用 TCP 连接还是关闭，继续请求其他资源。

## 关键点

- HTTP 是应用层请求-响应协议，所有 Web API 和页面资源获取都建立在它之上；理解它才能解释前后端如何通信。
- HTTP 报文由起始行、头部、空行和可选正文组成，头部承载缓存、认证、内容协商等关键语义，是排查问题的核心入口。
- HTTP 本身无状态，每个请求独立；Cookie/Token 等机制在应用层补充状态，理解这一点是设计会话管理和鉴权方案的基础。
- HTTP/1.1 引入持久连接和 Host 头，前者减少连接建立开销，后者支持同一服务器托管多个域名。
- 状态码分五类（1xx-5xx），其中 2xx 成功、3xx 重定向、4xx 客户端错误、5xx 服务端错误，是接口调试的第一判断依据。
- HTTP/2 和 HTTP/3 用二进制分帧、多路复用和 QUIC 解决 HTTP/1.1 的队头阻塞和头部冗余问题，代表了性能演进方向。

## 对比与权衡

- 相比 HTTP/1.0 的每次请求新建 TCP 连接，HTTP/1.1 的持久连接在减少握手和慢启动开销上更好，但仍存在同一连接上请求串行导致的队头阻塞问题。
- 相比 HTTP/1.1 的文本头部和串行响应，HTTP/2 的二进制分帧与多路复用在并发传输和头部压缩上更好，但协议实现和调试复杂度更高，且实际多强制使用 TLS。
- 相比 HTTPS，HTTP 明文传输在部署简单和性能损耗上更好，但在数据加密、身份认证和防篡改上远不如 HTTPS，因此敏感场景必须使用 HTTPS。

## 自测问题

**问: HTTP/1.0 和 HTTP/1.1 的主要区别是什么？**

HTTP/1.1 引入持久连接、Host 头、更完善的缓存协商(ETag/If-None-Match)、范围请求、分块传输编码等；持久连接减少握手，Host 头解决虚拟主机问题。回答时结合实际抓包报文说明。

**问: HTTP 是无状态协议，为什么网站还能记住登录状态？**

协议层不保存状态，但通过 Cookie 携带 Session ID 或 JWT，在服务端存储或令牌自包含实现应用层状态；还可提到 LocalStorage/Authorization 头等方式，说明为什么无状态利于扩展。

**问: HTTP 报文结构包含哪些部分？常用头部有哪些？**

起始行（请求行或状态行）、头部、空行、正文；请求头如 Host、Accept、Authorization、Cache-Control；响应头如 Content-Type、Set-Cookie、ETag、Location；要说明头部是元数据，控制缓存/认证/内容协商。

**问: HTTP/2 和 HTTP/3 是如何解决 HTTP/1.1 的性能问题的？**

HTTP/2 通过二进制分帧、多路复用、头部压缩 HPACK 解决应用层队头阻塞和头部冗余；HTTP/3 基于 QUIC，用 UDP 传输减少握手 RTT，并在传输层避免 TCP 队头阻塞。

**问: 从输入 URL 到页面展示，HTTP 在其中扮演什么角色？**

URL 解析后浏览器通过 HTTP/HTTPS 向服务器请求 HTML，解析后继续按需发起 CSS/JS/图片等 HTTP 请求；期间涉及 DNS、TCP、TLS、缓存、CDN、渲染等，HTTP 是资源获取的主干协议。

## 适用场景

- 设计或对接 RESTful API 时，需要根据 HTTP 方法、状态码和头部定义接口语义。
- 使用抓包工具（如 Chrome DevTools、Wireshark）排查接口错误、重复请求和缓存失效问题。
- 做 Web 性能优化时，通过持久连接、缓存策略、压缩和 CDN 减少 HTTP 请求成本。
- 面试复习网络基础时，把 HTTP 作为理解应用层协议和 Web 体系的核心入口。

## 标签

`HTTP` `网络协议` `Web基础` `请求-响应模型` `前后端联调`

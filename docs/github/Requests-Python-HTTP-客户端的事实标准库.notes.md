# Requests：Python HTTP 客户端的事实标准库

*原文: [https://github.com/psf/requests](https://github.com/psf/requests) · 来源: github · 生成时间: 2026-09-17T02:49:36.891132+00:00*

## 背景

在 Requests 出现前，Python 标准库 urllib/urllib2 接口割裂、API 不直观，发送带 query、cookie、表单编码、HTTPS 验证的请求需要大量样板代码。Requests 由 Kenneth Reitz 创建，目标是让 HTTP 请求“对人类友好”，后来由 Python Software Foundation 维护。它底层基于 urllib3，隐藏了连接池、重试、TLS 等复杂细节。

## 痛点

直接用 urllib 处理 HTTP 时，开发者需要手动拼接 URL 查询字符串、编码表单、管理 Cookie 和重定向，稍不注意就会写出脆弱且难以维护的代码；HTTPS 证书验证和超时设置也容易被忽略，导致线上安全隐患或挂死。

## 解决办法

Requests 提供统一的 request/get/post 等高层 API，内部基于 urllib3 管理连接池与 Keep-Alive；Session 对象持久化 Cookie 和连接复用；Response 对象自动处理 gzip/deflate 解压、编码推断、JSON 解析等。开发者只需关注 URL、参数和鉴权，其余由库按浏览器习惯完成。

## 关键代码示例

```python
import requests

# 基础 GET，自动处理 TLS 和连接复用
r = requests.get('https://httpbin.org/basic-auth/user/pass', auth=('user', 'pass'))
print(r.status_code)       # 200
print(r.headers['content-type'])
print(r.json())            # 自动 JSON 解析

# Session 复用 Cookie / 连接池
s = requests.Session()
s.auth = ('user', 'pass')
r2 = s.get('https://httpbin.org/basic-auth/user/pass')
print(r2.text)
```

第一个示例展示 requests.get 一行发起 HTTPS 请求，auth 参数负责 Basic Auth；Response 对象提供 status_code、headers、json() 等直观接口。第二个示例通过 Session 持久化认证和连接，避免每个请求重新握手、重复设置 Cookie。底层 urllib3 的连接池自动复用 TCP 连接。

## 关键流程

1. 安装：python -m pip install requests
2. 发送请求：使用 requests.get/post/put 等方法并传入 params/data/json/auth 等参数
3. 检查响应：通过 Response 的 status_code、headers、encoding、text、json() 读取结果
4. 需要多次请求同一主机时创建 Session 复用 Cookie 和连接池

## 关键点

- Requests 将 HTTP 动词映射为同名函数（get/post/put/delete 等），API 直观且易记忆，极大降低上手成本。
- Response 对象自动根据 headers 推断编码并解码，避免中文等非 ASCII 内容乱码。
- Session 默认持久化 Cookie 并复用底层 urllib3 连接池，显著提升连续请求性能。
- 默认启用浏览器式 TLS 证书验证，避免中间人攻击，同时可通过 verify=False 关闭但通常不推荐。
- 支持流式下载和超时设置，处理大文件或不可靠网络时必须使用 stream=True 和 timeout 参数。

## 对比与权衡

- 相比标准库 urllib/urllib2，Requests 在 API 简洁性、自动处理重定向/Cookie/编码上更好，但作为第三方库需要额外安装；urllib 更底层、无外部依赖但不适合日常业务开发。
- 相比底层的 urllib3，Requests 在易用性上更好，但暴露的连接池、重试策略等高级控制较弱；urllib3 适合需要精细控制连接行为的场景。
- 相比 httpx/aiohttp，Requests 不支持原生异步和 HTTP/2，因此在同步脚本和传统后端中更简单稳定，但在高并发异步场景不如后两者。

## 面试可能会问

**问: Requests 和 urllib 有什么区别？为什么 Requests 更受欢迎？**

urllib 是标准库但 API 割裂，需手动处理编码、quote、Cookie、HTTPS 上下文等；Requests 基于 urllib3 提供统一高层 API，自动处理重定向、会话、解压、JSON 等，代码更少且更符合 Python 习惯。

**问: Session 对象的作用是什么？和直接 requests.get 有何不同？**

Session 内部维护 CookieJar 和连接池，可以跨请求保留 Cookie、复用 TCP/TLS 连接、设置默认 headers/auth；直接调用 requests.get 每次创建临时 Session，无法跨请求保持状态，性能也较差。

**问: 如何处理请求超时，避免程序无限挂起？**

使用 timeout 参数，可以传单个数值（连接+读取总超时）或元组 (connect_timeout, read_timeout)。生产环境必须设置超时，否则遇到无响应的服务器会一直阻塞。

**问: 如何下载大文件而不把整个内容读入内存？**

使用 stream=True 使响应体分块传输，然后用 iter_content(chunk_size) 逐块写入文件；也可以配合 Context Manager 关闭响应。这体现了 Requests 流式下载能力。

**问: Requests 是如何自动解析 JSON 的？如果服务端返回非 JSON 会怎样？**

Response.json() 内部使用标准库 json.loads 解析 r.text，并可根据响应头 charset 处理编码；若内容不是合法 JSON 会抛 json.JSONDecodeError，业务代码需要捕获。

## 适用场景

- 调用 RESTful API 或第三方 HTTP 接口，替代 curl/urllib 编写可维护的同步客户端。
- 写爬虫或自动化脚本，需要处理登录 Cookie、重定向、表单提交和文件上传。
- 在服务端 Python 应用中调用内部微服务，并配合 Session 连接池提升性能。
- 需要快速验证 HTTP 接口行为、编写测试用例或做接口冒烟测试。

## 标签

`Python` `HTTP` `Requests` `网络编程` `PyPI`

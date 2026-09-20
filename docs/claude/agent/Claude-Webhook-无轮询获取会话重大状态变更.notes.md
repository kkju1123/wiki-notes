# Claude Webhook：无轮询获取会话重大状态变更

*原文: [https://platform.claude.com/docs/en/managed-agents/webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks) · 来源: web · 生成时间: 2026-09-20T07:36:36.984235+00:00*

## 背景

Webhook是服务端到服务端的主动推送机制，用来解决轮询带来的高延迟和请求浪费。在Claude平台中，Agent会话是长时运行交互，流式实时内容已通过SSE事件流提供；但很多后台系统无法或不需要维持SSE长连接，因此只在重大状态变化（运行、等待、预算到达、终止等）时通过Webhook通知订阅方。

## 痛点

没有Webhook时，下游系统只能定时GET会话状态，既浪费配额也可能错过关键状态转变，且难以实时感知预算到达或自动重试等事件。若直接推送完整对象，重试时可能把陈旧数据当作最新状态；同时服务端若忽略签名验证、重复投递、乱序和自动禁用规则，容易漏处理或产生错误状态。

## 解决办法

核心做法是在Claude Console中注册一个公网HTTPS端点，并只订阅需要的事件类型。每个投递仅携带事件type和id，接收方先用SDK的unwrap()验证签名并解析事件，返回任意2xx确认，然后按id调用GET获取最新资源。签名使用whsec_密钥对webhook-id/timestamp做校验，超过5分钟的投递视为过期。投递会重试且可能重复，因此用event.id做幂等去重；事件顺序不保证，状态机必须基于拉取到的资源而不是事件到达顺序。返回3xx、URL解析到非公网IP或长期投递失败会导致端点被自动禁用，需修复后手动重新启用。

## 关键代码示例

```python
import os
from flask import Flask, request
from anthropic import unwrap  # SDK 提供的签名验证/解析辅助函数

app = Flask(__name__)
seen_event_ids = set()  # 生产环境应替换为 Redis 等共享去重存储

@app.post("/webhooks/claude")
def claude_webhook():
    # 1. 验证签名并解析事件；签名无效或超过5分钟会抛异常
    try:
        event = unwrap(
            request.get_data(),
            headers=request.headers,
            secret=os.environ["ANTHROPIC_WEBHOOK_SIGNING_KEY"],
        )
    except Exception:
        return "", 401

    # 2. 幂等去重：同一事件可能因重试投递多次
    if event.id in seen_event_ids:
        return "", 200
    seen_event_ids.add(event.id)

    # 3. 事件只有 type/id，状态以 GET 到的资源为准
    if event.data.type == "session.status_run_started":
        resource = fetch_session(event.data.id)  # GET /v1/sessions/{id}
        update_state(resource)

    return "", 204
```

这段代码先调用SDK的unwrap()对请求体、请求头中的webhook-id/timestamp和signature进行校验，并检查投递是否在5分钟内，签名无效或过期直接返回401。通过内存集合标记已处理的event.id实现幂等去重，因为同一事件的每次重试投递都携带相同event.id，生产环境应换成Redis等共享存储。随后根据data.type分支处理业务，事件负载不携带完整对象，因此需要调用GET获取最新资源，再更新本地状态。最后返回204确认投递成功。

## 关键流程

1. 在Claude Console的 Manage > Webhooks 中创建端点，填写公网HTTPS URL并选择要订阅的事件类型
2. 创建后保存仅显示一次的 whsec_ 签名密钥到安全存储，并配置到服务端环境变量
3. 接收投递时先用SDK unwrap()验证 webhook-id/webhook-timestamp/webhook-signature 并解析事件
4. 先返回任意2xx确认；返回3xx或其他非2xx会计入失败或触发自动禁用
5. 基于event.id做幂等去重，再按 data.type 分支并用 data.id 去GET完整资源
6. 监控投递失败率；端点被自动禁用后修复问题并在Console重新启用

## 关键点

- Webhook只推送事件type和id，不推送完整对象，接收方必须GET资源；这样重试不会投递陈旧数据，也保持每次投递体积小。
- 签名验证使用whsec_密钥和请求头中的 webhook-id、webhook-timestamp、webhook-signature，SDK unwrap()会同时校验签名和5分钟新鲜度，防止伪造和重放。
- 返回任意2xx表示确认，其他状态会计入交付失败；尤其3xx会立即禁用端点且不会跟随重定向，防止投递链路失控。
- event.id按事件唯一而不是按投递唯一，重试投递值相同，因此用event.id做幂等去重即可。
- 事件顺序不保证，idled可能先于outcome结束到达，deleted可能先于archived；状态机必须基于GET到的资源，而不是事件到达顺序。
- Webhook不是持久日志，三次投递失败后丢弃且不补发；若需要观察每一次状态变迁，应通过API定期list/fetch对账。

## 对比与权衡

- 相比SSE事件流，Webhook不需要订阅方维持长连接，更适合服务端后台集成和低频状态通知；但SSE能实时传递流式增量内容，Webhook仅通知重大状态且不携带完整对象。
- 相比轮询，Webhook由平台主动推送，延迟低且省请求配额；但需要部署公网HTTPS端点并处理签名、重试、幂等和自动禁用等额外可靠性问题。
- 与通用消息队列或持久事件流相比，Claude Webhook是尽力投递，最多三次后丢弃且无长时间持久化；如果要求不丢事件，需要API对账，不能只依赖Webhook。

## 自测问题

**问: 为什么Webhook事件不直接带完整对象，只带type和id？**

避免重试时把过期数据当作最新状态；保持每次投递体积小；强制接收方按需GET最新资源，数据更新鲜，减少陈旧数据处理。

**问: 如果同一个event.id收到两次，应该怎么处理？**

这是重复投递，不是新事件；用event.id做幂等去重，处理前先检查是否已处理过；重复投递也返回2xx确认，不要返回错误。

**问: 事件顺序乱序怎么办？例如deleted先于archived到达。**

不要依赖事件到达顺序驱动状态；以GET到的资源当前状态为准；对已删除资源做特殊处理；可用API list/fetch进行周期性对账。

**问: 为什么3xx会被禁用且不跟随重定向？**

防止webhook投递被重定向到不可控地址，避免安全风险（如SSRF）或投递到内网；如果端点迁移，应当在Console更新URL并手动重新启用。

**问: Webhook漏了事件怎么办？**

Webhook最多三次重试后丢弃，不补发；它不是持久日志，不保证不丢。需要不漏事件时，不能只依赖Webhook，应通过定期调用API list/fetch做reconciliation，并在需要事件前提前订阅，因为后续订阅不会backfill。

## 适用场景

- 服务端需要感知会话状态变化并触发下游动作，例如启动、等待输入、终止后归档
- 预算达到或会话自动重试/终止时的告警通知
- 多Agent线程生命周期监控，追踪coordinator、child thread或advisor的创建、等待、终止
- 审计或状态同步场景，但不能依赖Webhook作为唯一可靠事件源，需要配合API对账

## 标签

`Webhook` `Claude API` `事件驱动` `签名验证` `可靠性设计`

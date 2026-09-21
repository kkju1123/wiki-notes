---
title: 1. `stop_reason` 到底是什么？
url: wikibar://summary/summary/1-stop_reason-到底是什么
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T01:42:18.568405+00:00'
---

#claude stop reason

可以把这页理解成：**Claude 每次 API 返回时都会告诉你“我为什么停下来了”**，这个字段就是 `stop_reason`。你的程序不能只拿 `response.content` 就完事，最好根据 `stop_reason` 决定下一步。([Claude Platform][1])

## 1. `stop_reason` 到底是什么？

例如：

```python
response = client.messages.create(...)

print(response.stop_reason)
```

可能得到：

```text
end_turn
```

它不是报错，而是在告诉你的程序：

> “我这次为什么停止生成？”

所以整体思路是：

```text
请求 Claude
   ↓
Claude 返回 response
   ↓
检查 response.stop_reason
   ↓
决定下一步
```

官方强调，`stop_reason` 属于**成功的 API response**；真正的请求错误一般才是 HTTP `4xx / 5xx`。([Claude Platform][1])

---

## 2. `end_turn`：正常说完了

最普通的情况：

```text
stop_reason = "end_turn"
```

意思就是：

> Claude：我回答完了。

例如：

```text
User: 1+1 等于多少？

Claude: 2。
        ↑
     end_turn
```

这种情况直接使用答案即可。([Claude Platform][1])

不过有个小坑：偶尔可能出现 `end_turn` 但内容几乎为空，尤其是 Tool Calling 后消息结构写得不对。官方特别提醒：**不要在 `tool_result` 后面紧接额外文字**；如果结构修正后仍为空，可以下一轮单独发 `"Please continue"`。([Claude Platform][1])

---

## 3. `max_tokens`：字数额度用完了

例如：

```python
response = client.messages.create(
    max_tokens=100,
    ...
)
```

Claude本来还想继续：

```text
LLM 是一种大型语言模型，它首先……
                              ↑
                           被截断
```

此时：

```text
stop_reason = "max_tokens"
```

意思不是 Claude 认为自己讲完了，而是：

> “你允许我生成的 token 数已经用完了。”

可以提高 `max_tokens`，或者设计继续生成逻辑。

特别注意：如果刚好在生成 `tool_use` 时被截断，那么工具调用可能是不完整的，**不能直接执行这个残缺的 tool call**，应该提高 `max_tokens` 后重新请求。([Claude Platform][1])

---

## 4. `stop_sequence`：撞到了你设置的停止词

你可以自己设置：

```python
stop_sequences=["END", "STOP"]
```

如果 Claude 生成过程中碰到：

```text
今天我们介绍神经网络 END
```

就停止：

```text
stop_reason = "stop_sequence"
```

还可以通过：

```python
response.stop_sequence
```

知道具体是哪个停止序列触发的。([Claude Platform][1])

---

## 5. `tool_use`：Claude 在等你执行工具

这个和我们上一轮讲的正好接起来。

假设 Claude 惏查天气：

```text
User
 ↓
Claude
 ↓
"我要调用 get_weather"
 ↓
stop_reason = tool_use
```

返回类似：

```json
{
  "type": "tool_use",
  "name": "get_weather",
  "input": {
    "location": "Basel"
  }
}
```

你的程序需要：

```text
找到 tool_use
↓
执行 get_weather
↓
获得结果
↓
作为 tool_result 返回 Claude
↓
Claude 继续生成
```

所以：

```text
tool_use ≠ Claude回答完了

tool_use = Claude暂停，等你的工具结果
```

如果同时存在 `server_tool_use`，可能是 Anthropic 自己负责执行的服务器工具；而你自己的 `tool_use` 仍然需要客户端处理。Programmatic Tool Calling 下，`tool_use` 还可能是已经运行中的 `code_execution` 发起的，此时返回 `tool_result` 是**恢复暂停的代码**。([Claude Platform][1])

---

## 6. `pause_turn`：服务器工具干太久，先暂停

这个和 `tool_use` 很容易混。

例如 Claude 自己在使用服务器端：

```text
web_search
↓
分析
↓
web_search
↓
分析
↓
……
```

Anthropic 的服务器工具循环默认达到 **10 次迭代上限**时，可能返回：

```text
stop_reason = "pause_turn"
```

人话：

> “这一轮服务器工具还没真正做完，但这个 API 请求先到这里，你把我的 response 发回来，我继续。”

所以：

```text
tool_use
→ 等你的客户端工具结果
→ 你返回 tool_result

pause_turn
→ 服务器端工具循环暂停
→ 把 assistant response 原样带回下一请求继续
```

这是非常重要的区别。([Claude Platform][1])

---

## 7. `refusal`：Claude 拒绝回答

例如：

```text
stop_reason = "refusal"
```

代表请求触发了拒绝机制。

一个容易误解的地方是：**它仍然可能是 HTTP 200。**

也就是说：

```text
HTTP 200
≠ Claude一定回答了你的问题
```

因为 HTTP 200 只是：

> API 请求成功处理。

而：

```text
stop_reason = refusal
```

表示：

> Claude 成功处理请求，但决定拒绝生成该内容。

这时还可以查看 `stop_details`；官方文档说明，其他 stop reason 下 `stop_details` 通常为 `null`。([Claude Platform][1])

---

## 8. `model_context_window_exceeded`：上下文窗口塞满了

这个和 `max_tokens` 又不一样。

假设模型总上下文容量可以粗略理解成：

```text
历史聊天
+ System Prompt
+ Tool 信息
+ 当前问题
+ Claude 输出
----------------
≤ Context Window
```

如果整个 context window 满了：

```text
stop_reason =
"model_context_window_exceeded"
```

意思就是：

> 不是你设置的 `max_tokens` 先用完，而是整个模型上下文空间已经装不下了。

因此：

```text
max_tokens
→ 你给“输出”设置的上限到了

model_context_window_exceeded
→ 模型整个上下文窗口满了
```

这种 response 仍然有效，只是被截断。([Claude Platform][1])

---

## 9. 实际开发怎么写？

核心思想其实就是一个分支：

```python
if response.stop_reason == "end_turn":
    # 正常完成
    use_answer()

elif response.stop_reason == "tool_use":
    # 执行客户端工具，并返回 tool_result
    handle_tools()

elif response.stop_reason == "pause_turn":
    # 把 assistant response 带回去继续
    continue_turn()

elif response.stop_reason == "max_tokens":
    # 输出被截断
    handle_truncation()

elif response.stop_reason == "model_context_window_exceeded":
    # 上下文窗口满了
    handle_context_limit()

elif response.stop_reason == "refusal":
    # 处理拒绝
    handle_refusal()

elif response.stop_reason == "stop_sequence":
    # 命中了自定义停止词
    handle_stop_sequence()
```

所以 `stop_reason` 可以理解成**Claude 给你程序发的“下一步行动信号”**。官方也建议应用始终检查这个字段。([Claude Platform][1])

## Cheatsheet

| `stop_reason`                   | 人话            | 程序怎么办                       |
| ------------------------------- | ------------- | --------------------------- |
| `end_turn`                      | ✅ 我正常说完了      | 直接使用答案                      |
| `max_tokens`                    | ✂️ 输出额度用完，被截断 | 增大 `max_tokens` / 继续生成      |
| `stop_sequence`                 | 🛑 碰到你设置的停止词  | 检查 `response.stop_sequence` |
| `tool_use`                      | 🔧 我要调用你的工具   | 执行工具 → 返回 `tool_result`     |
| `pause_turn`                    | ⏸️ 服务器工具还没跑完  | 把 assistant response 带回去继续  |
| `refusal`                       | 🚫 我拒绝回答      | 看 `stop_details`，按需处理/回退    |
| `model_context_window_exceeded` | 📦 整个上下文窗口满了  | 当作截断处理，缩减 context           |

最值得背的是：

```text
end_turn     = 真说完了
max_tokens   = 输出被截断
tool_use     = 等你的工具
pause_turn   = 服务器工具暂时停一下，继续这一轮
refusal      = 拒绝
context...   = 整个上下文塞满
```

以及最容易考/最容易写错的一组：

```text
tool_use   → 你执行客户端工具 → 返回 tool_result

pause_turn → 不需要你造 tool_result
           → 把 Claude 当前 response 带回去
           → 让服务器工具继续跑
```

[Claude 官方：Handling stop reasons](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons "停止原因与回退 - Claude Platform Docs"
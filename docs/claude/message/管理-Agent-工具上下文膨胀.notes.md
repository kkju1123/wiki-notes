# 管理 Agent 工具上下文膨胀

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context](https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context) · 来源: web · 生成时间: 2026-09-20T04:09:04.425127+00:00*

## 背景

在大模型 Agent 中，工具定义（schema）和每次 tool_result 都会进入上下文窗口；当工具集很大或任务轮次很长时，上下文会迅速膨胀，导致成本上升、延迟增加，甚至任务中断。这类问题在 Claude 等平台的 long-running agents 中尤其突出。因此需要一组在工具调用管线不同阶段控制上下文占用的方法。

## 痛点

没有这些手段时，大量无关工具 schema 会在每轮都占用 token，而链式工具调用的中间结果会被反复写入历史；长会话最终可能撑爆 context window。结果是要么强制截断丢失关键信息，要么任务失败，同时 token 成本显著增加。

## 解决办法

按 token 流向分阶段治理：tool search 像“延迟加载”，只在需要时把工具定义查进上下文；programmatic tool calling 像“批量执行”，把多次工具调用合并为沙箱脚本，只回传最终输出；prompt caching 像“缓存复用”，缓存固定 tool definitions 前缀，降低重复请求成本；context editing 像“垃圾回收”，删除不再相关的旧 tool_result。四者互相补充，可组合使用。

## 关键代码示例

```python
tools = [{
    'name': 'execute_python',
    'description': 'Run Python in a sandbox with get_weather/search_hotels',
    'input_schema': {'type': 'object', 'properties': {'code': {'type': 'string'}}}
}]
# Claude 生成的脚本
code = '''
import json
cities = ['Paris', 'London', 'Tokyo']
out = {}
for c in cities:
    out[c] = {
        'weather': get_weather(c),
        'hotels': search_hotels(c)[:3]
    }
print(json.dumps(out))
'''
# 执行后仅把 print 输出作为 tool_result 返回
```

代码先注册一个 execute_python 工具，而不是把 get_weather、search_hotels 等多个工具 schema 都塞进上下文。Claude 生成的脚本在沙箱内循环调用预置函数，中间天气和酒店结果只存在于脚本变量中，不会形成多条 tool_result。最终只有 print 输出返回给模型，对应 programmatic tool calling 的原理：用一次工具调用替代多次往返，从源头压缩上下文。

## 关键流程

1. 第 1 步：先给稳定的工具定义启用 prompt caching，从第一天起降低重复请求的 token 成本。
2. 第 2 步：当工具集超过约 20 个或基础上下文占用明显时，引入 tool search 按需加载工具定义。
3. 第 3 步：当单次对话足够长、早期结果不再相关时，加入 context editing 清理旧 tool_result。
4. 第 4 步：如果发现大量重复的小工具调用链，可改用 programmatic tool calling 合并为单个脚本执行。

## 关键点

- 工具定义和 tool_result 是 Agent 上下文膨胀的主要来源，治理前应先定位 token 流向。
- Tool search 通过按需发现工具定义来降低基础上下文，适合 20+ 工具集，但会增加一次查找往返。
- Programmatic tool calling 把多步工具调用压缩成一个沙箱脚本，只有最终输出进入上下文，能显著减少轮次和 token。
- Prompt caching 不减少上下文 token 数，但能大幅降低固定工具定义在重复请求中的计费成本。
- Context editing 删除长对话中已经无用的 tool_result，相当于对历史做垃圾回收，避免重启对话丢失关键信息。
- 四种方法可以安全组合，各解决定义加载、往返次数、重复成本和历史残留四个不同问题。

## 对比与权衡

- 相比 tool search，prompt caching 保留了完整工具定义，请求时不会增加额外查找延迟，但它不减少上下文 token 数，只降低重复请求成本。
- 相比 programmatic tool calling，context editing 是在历史中事后删除旧结果，仍允许模型在每一步观察中间输出；programmatic 则从源头不产生中间 tool_result，但需要沙箱执行环境，且中间过程不可见。
- 相比直接重启或暴力截断对话，context editing 能保留仍重要的上下文，同时删除已失效的 tool_result，避免任务状态丢失。

## 自测问题

**问: 长运行 Agent 上下文膨胀，你会从哪些角度优化？**

先分析 token 主要来自工具定义、中间 tool_result 还是重复请求；然后分别用 tool search 按需加载定义、programmatic tool calling 合并调用链、context editing 清理历史、prompt caching 降低固定前缀成本。强调这些手段可以组合。

**问: Prompt caching 和 context editing 有什么区别？**

Prompt caching 优化的是成本，不减少上下文内容，适合固定工具定义在高请求量下复用缓存前缀；context editing 优化的是上下文长度，直接删除旧结果，降低每轮输入 token 和延迟。两者目标不同，可以同时用。

**问: Tool search 会增加一次查找往返，为什么还要用？**

当工具有 50 个甚至更多时，把所有 schema 每轮都放入上下文会占用大量 token，并分散模型注意力。一次查找往返的成本通常远小于每轮携带全量 schema 的成本，尤其适合大多数轮次只需要少量工具的场景。

**问: Programmatic tool calling 有什么适用条件和风险？**

适合一连串确定性较强、不需要每步根据中间结果做复杂决策的工具调用。风险包括中间结果不可见、脚本错误难以单步调试、沙箱安全边界和超时控制。需要可观测性与回退策略。

**问: 这四个方法如何在一个高吞吐 Agent 中落地？**

先启用 prompt caching 覆盖稳定工具定义；当工具数量超过约 20 个时加 tool search；会话变长后加 context editing；如果出现重复的小工具链，再考虑 programmatic tool calling。分阶段引入并监控 token 使用。

## 适用场景

- 工具集超过 20 个、每轮只用到少数工具的企业数据平台 Agent。
- 需要长时间多轮调用工具的研究、客服或自动化运维 Agent。
- 工具定义固定且请求量大的 API 产品，用于降低 token 成本。
- 可预测的工具链任务，例如抓取数据、批量查询或生成报表。

## 标签

`AI Agent` `上下文管理` `Tool Calling` `Prompt Caching` `LLM 工程`

# 工具参考：Anthropic 内置工具与客户端工具集

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference) · 来源: web · 生成时间: 2026-09-20T04:07:11.029491+00:00*

## 背景

Claude 模型本身只能生成文本，无法直接获取实时信息、执行代码或操作图形界面。工具调用机制让模型在推理过程中向应用发起结构化请求，由外部系统执行并把结果回传。Anthropic 提供了一批预定义工具并统一了版本管理，以减少集成成本、避免每个团队重复造轮子。

## 痛点

如果不清楚服务端工具和客户端工具的执行位置，容易把运行责任搞错，导致安全或架构问题。配置客户端工具集时若忽略 defer_loading 与 cache_control 的互斥规则，可能造成工具不加载或缓存失效。处理 tool_use 块时只按 name 分发，可能因为不同工具集重名而调用错误。

## 解决办法

所有工具都通过请求中的 tools 数组声明，每个条目用 type 标识具体工具版本。服务端工具由 Anthropic 托管执行，应用只负责传入参数和接收结果；客户端工具由 Anthropic 定义 schema，但应用自己执行。客户端工具集用一个入口声明一批固定成员，应用通过 configs 控制成员启用和延迟加载，并根据 toolset_name 与 name 的组合分发调用。版本后缀 YYYYMMDD 用于管理演进，旧版本通常保留以保证兼容，选择时看能力需求和目标模型，而不是盲目升级。

## 关键代码示例

```json
[
  {"type": "web_search_20260318", "name": "web_search"},
  {
    "type": "computer_toolset_20260801",
    "configs": {
      "screenshot": {"enabled": true, "defer_loading": false},
      "click": {"enabled": true, "defer_loading": false}
    },
    "cache_control": {"type": "ephemeral"},
    "allowed_callers": ["direct"]
  }
]
```

这段 JSON 是请求 tools 数组的片段。web_search 是服务端工具，仅需 type 和 name；computer_toolset 是客户端工具集，入口不写 name，通过 configs 启用 screenshot 和 click 两个成员，并可单独设置 cache_control。allowed_callers 限制为 direct，表示只允许模型直接调用。

## 关键流程

1. 根据任务需求选择工具：实时信息用 web_search，抓取页面用 web_fetch，执行代码用 code_execution，操作 GUI 用 computer/browser 工具集。
2. 在 tools 数组中声明服务端工具、客户端工具或客户端工具集，注意不同工具的 type 版本后缀。
3. 对客户端工具集设置 configs：按成员名配置 enabled 和 defer_loading，不能禁用全部成员，也不能在成员配置里放其他字段。
4. 如需 prompt caching，把 cache_control 放在非延迟工具或工具集入口；若成员全部 defer，则不能在入口设 cache_control。
5. 处理响应中的 tool_use 块：读取 toolset_name 和 name，根据这对标识分发到具体执行逻辑，执行后返回 tool_result。
6. 如果同一回合有多个成员调用，按顺序执行 batch action，并将结果一次性回传。

## 关键点

- Anthropic 工具分为服务端工具和客户端工具，执行位置不同，前者由 Anthropic 托管，后者由应用负责执行，架构设计必须区分。
- 工具类型字符串带有 YYYYMMDD 日期后缀，新版本发布后旧版本仍保留，选择版本取决于能力需求和目标模型，不是越新越好。
- 客户端工具集是一个入口声明一组固定成员，入口无 name 字段，成员名称、描述和 schema 由 Anthropic 预定义，应用负责执行所有调用。
- configs 中的 defer_loading 必须对所有启用的成员统一设置，因为工具搜索下工具集作为整体加载和扩展，混用会导致行为不一致。
- 处理成员调用时必须同时看 toolset_name 和 name，因为不同工具集可能共享成员名，例如多个工具集都有 screenshot。
- 缓存的 breakpoint 不能落在 deferred 定义上，因此 defer_loading 和入口 cache_control 不能同时使用，否则缓存前缀不包含工具定义。

## 对比与权衡

- 相比服务端工具，客户端工具在灵活性上更高，应用可以自定义执行环境和安全策略，但需要自己处理运行时、错误和超时问题。
- 相比普通单工具声明，客户端工具集用一个入口打包多个相关工具，减少了 tools 数组的冗余，但配置粒度较粗，成员只能统一设置 defer_loading。
- 相比 legacy/successor 这类线性版本关系，capability-keyed 和 variant 版本允许新旧方案并存，按需选择而非强制迁移，更适合渐进式升级。
- tool_search_tool_regex 与 tool_search_tool_bm25 是两种并存的工具检索算法，regex 适合模式匹配，bm25 适合关键词相关性排序，选择取决于工具库规模和命名规范。

## 自测问题

**问: Anthropic 的工具分为哪几类？执行责任分别由谁承担？**

分为服务端工具、客户端工具和客户端工具集。服务端工具在 Anthropic 基础设施上执行，应用只传参和收结果；客户端工具由 Anthropic 定义 schema，但应用执行；客户端工具集是预定义的客户端工具集合，一个入口代表多个成员，应用同样负责执行。

**问: 为什么工具类型字符串要带日期后缀？实际项目中如何选择版本？**

日期后缀用于标识工具行为、schema 或模型支持的变化，旧版本保留以避免破坏现有集成。选择时先看目标模型支持哪些版本，再看是否需要新能力，如动态内容过滤、缓存绕过或程序化调用，最后考虑稳定版与 beta 版的取舍。

**问: 客户端工具集的 configs 里，defer_loading 和 cache_control 为什么不能同时放在入口？**

deferred 定义不属于缓存前缀，如果入口设置 cache_control，断点会落在不包含工具定义的位置，导致缓存不完整或失效。正确做法是让所有启用成员统一 defer，并在同一请求中声明一个非延迟的工具搜索工具来暴露工具集，把 cache_control 放在那个非延迟工具上。

**问: 收到 computer 工具集的 tool_use 块后，应该怎样分发？**

读取 tool_use 中的 toolset_name 和 name，用这对组合分发，因为自定义工具可能重名，且不同工具集之间也可能共享 member 名。多个成员调用在同一回合形成 batch action，需要按顺序执行，并把每个结果以对应的 tool_result 回传。

**问: 如果我想让一个客户端工具集只对模型直接调用开放，应该设置什么？**

在该工具集条目上设置 allowed_callers 为 ["direct"]，表示只允许模型直接发起调用，不允许其他工具或流程间接调用。需要注意该字段只接受 direct 值，并且不是所有环境都支持，需结合具体 API 版本。

## 适用场景

- 构建需要实时信息的研究助手或新闻摘要 Agent，通过 web_search 和 web_fetch 获取最新内容。
- 在沙箱中执行用户代码并返回结果，适用于代码解释器或数据分析场景。
- 自动化桌面或浏览器操作，例如用 computer use 工具集完成 GUI 流程，用 browser use 工具集完成 Web 任务。
- 为 Claude 应用增加持久记忆、Shell 执行或文件编辑能力，适合本地开发或运维 Agent。

## 标签

`Claude Tools` `Tool Use` `Anthropic API` `Client Toolsets` `Agent 开发`

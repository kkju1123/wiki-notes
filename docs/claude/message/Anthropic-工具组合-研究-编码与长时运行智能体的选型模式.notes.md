# Anthropic 工具组合：研究、编码与长时运行智能体的选型模式

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-combinations](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-combinations) · 来源: web · 生成时间: 2026-09-20T04:11:14.435964+00:00*

## 背景

LLM 本身无法实时搜索、执行代码、操作系统或跨会话记忆，因此 Anthropic 把能力拆成 web_search、code_execution、text_editor、bash、memory、browser_use 等工具。单个工具只覆盖一类动作，现实任务通常需要多个阶段串起来；工具组合的本质是把智能体工作流拆成可复用的互补环节。这类似 Unix 管道：一个工具负责获取/发现，另一个负责处理/执行/验证。

## 痛点

不懂组合的人容易把工具设计成孤立调用：要么抓取所有页面导致 token 和延迟爆炸，要么只改代码不跑测试形成不了闭环，要么跨会话丢失用户上下文。更糟的是在简单任务上直接上 computer_use，导致慢且不稳定。合理组合可以明显降低失败率和成本。

## 解决办法

核心是互补阶段配对：一个工具做发现/获取，另一个做执行/验证；memory 作为正交横切负责持久化。研究型用 web_search 获取实时资料，code_execution 在服务端跑 Python 做统计、聚合、可视化；编码型用 text_editor 改文件，bash 跑测试/构建形成 edit-test loop；搜索+fetch 采用先看 snippet 再按需抓取完整页面，避免盲目 fetch。浏览器/桌面工具是兜底：任务在网页内优先 browser_use，需要任意 GUI 才用 computer_use。类比团队分工：研究员先查资料再算数据，工程师改完代码要跑 CI，记忆是团队笔记。

## 关键代码示例

```python
# 以下为示意：核心是 tools 数组里的搭配，具体 type 以官方 catalog 为准

# 1. 研究型：先搜索实时信息，再用服务端代码执行分析/制表
research_tools = [
    {'type': 'web_search'},
    {'type': 'code_execution'},
]

# 2. 编码型：编辑文件 + 运行测试，形成 edit-test loop
coding_tools = [
    {'type': 'text_editor'},
    {'type': 'bash'},
]

# 3. 长时运行：memory 与其他工具并列，负责跨会话持久化
support_tools = [
    {'type': 'memory'},
    {'type': 'web_search'},
]

# 4. 浏览器内任务优先用 browser_use；computer_use 只是兜底
browser_tools = [
    {'type': 'browser_use'},
]
```

这段代码只展示 tools 数组：每个工具用 type 声明，Claude 根据声明选择调用。组合原则是让前一个工具的输出成为下一个工具的输入：search 得到 URL，code_execution 分析数据；text_editor 修改文件，bash 验证修改。memory 不与 web_search 冲突，而是正交地保存/检索事实。实际请求中 tools 数组要放进 messages.create 的 tools 参数，并配合 tool_use 处理循环。

## 关键流程

1. 分析任务需要的信息与动作：是否要实时数据、计算、文件修改、跨会话状态或 GUI 操作。
2. 如果有实时数据且需要计算，选 web_search + code_execution；搜索结果不足时再补一轮搜索。
3. 如果任务是软件开发，选 text_editor + bash：读码、改文件、跑测试，必要时限制目录与命令白名单。
4. 如果答案依赖长文页面，选 web_search + web_fetch：先看 snippet，再只 fetch 2-3 个相关页面。
5. 如果 agent 需要跨会话记忆，把 memory 与上述任意工具并列加入 tools 数组。
6. 如果任务是网页表单/阅读/多标签操作，选 browser_use；只有需要任意桌面 GUI 时才升级到 computer_use。

## 关键点

- 工具组合的核心是互补阶段，而不是堆砌功能：一个工具负责发现/获取，另一个负责处理/验证，这样职责边界清晰，失败可定位。
- web_search + code_execution 同时解决‘信息新鲜度’和‘计算可信度’：模型负责筛选与生成代码，服务端执行保证可复现，适合财务/统计类问题。
- text_editor + bash 是软件工程闭环：只改代码不跑测试会积累错误；但 bash 是客户端执行，必须配合受限工作目录和命令白名单。
- search + fetch 的精髓是‘先看摘要再抓全文’，避免无条件抓取所有结果；这能显著节省 token 和延迟，但可能遗漏靠后结果，需要迭代搜索。
- memory 是正交横切能力：它不改变其他工具行为，只提供跨会话事实存取，解决上下文窗口重置导致的遗忘问题。
- computer_use 最通用但最慢、成本最高；优先选择更窄的工具，browser_use 在网页任务中更稳，只有需要驱动任意桌面应用才用 computer_use。

## 对比与权衡

- 相比只使用 web_search，web_search + code_execution 在数据处理、计算和可视化上更强，但需要模型额外生成代码，失败的调试成本也更高。
- 相比直接抓取搜索结果的所有 URL，web_search + web_fetch 在 token 消耗和响应延迟上明显更优，但可能因为只 fetch 前几个结果而漏掉靠后相关信息。
- 相比只使用文本编辑工具，text_editor + bash 能形成修改-测试闭环、更接近真实开发流程，但在安全上更危险，必须做沙箱、目录限制和命令白名单。
- 相比 computer_use，browser_use 在纯网页任务上更稳定、能基于元素引用操作，但在非浏览器桌面应用上覆盖不了。

## 自测问题

**问: 为什么推荐 web_search + code_execution，而不是让模型在上下文里直接算？**

模型长文本推理容易算错统计量，也不能处理大量数据或画出可靠图表；code_execution 用 Python 等确定性地执行，结果可复现、可检查。搜索保证数据实时，执行负责分析，两者解耦让 agent 可以‘查完再算、算中再查’。

**问: 给 coding agent 配 bash 工具时，如何避免安全问题？**

bash 是客户端执行，能触碰宿主机，因此要最小权限：限制 working directory、用命令 allowlist/denylist、设置超时、禁网络或仅允许内部包源，最好在容器/隔离沙箱中运行并记录审计日志；同时不要把敏感凭据写进可访问文件。

**问: web_search + web_fetch 与直接抓取所有搜索结果的区别？**

前者让 Claude 先读搜索摘要候选，再只 fetch 2-3 个相关页面，降低 token 和延迟，避免把大量无关正文塞进上下文；代价是可能漏掉低排名但重要的结果，因此要支持‘发现缺口再搜索’的循环。

**问: memory 工具和其他工具是什么关系？为什么说它是正交的？**

memory 负责跨会话持久化事实，其他工具负责当前任务的具体动作；它不改变搜索/执行/编辑工具的行为，只是给模型一个写入和检索长期记忆的位置，解决上下文窗口重置后失忆。设计上通常把 memory 也声明在 tools 数组里，由模型判断何时存取。

**问: 什么时候用 browser_use，什么时候用 computer_use？**

任务完全在浏览器内，如表单填写、阅读网页、多标签操作，优先 browser_use，因为它可基于元素引用和页面成员工具更稳定；任务需要跨桌面应用、无 API 的遗留软件或视觉验证时，才用 computer_use。computer_use 最通用但最慢，因每批操作后常需新截图，成本高。

## 适用场景

- 金融/商业研究：对比各云厂商季度营收、抓取财报并计算指标。
- 自动化编码：修 bug、重构、跑测试/构建的开发闭环。
- 长文档采集与引用：搜索技术规范/法律文本，只 fetch 相关页面并引用具体段落。
- 跨会话客服/项目助理：记住用户偏好、历史问题与项目决策。

## 标签

`Anthropic` `tool use` `智能体` `工具组合` `Claude`

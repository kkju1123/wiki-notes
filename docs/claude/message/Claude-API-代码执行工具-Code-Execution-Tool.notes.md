# Claude API 代码执行工具（Code Execution Tool）

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) · 来源: web · 生成时间: 2026-09-20T03:52:01.744326+00:00*

## 背景

大型语言模型基于概率生成文本，对需要精确计算、长程逻辑推导或真实文件解析的任务极易出错。Claude 代码执行工具的出现，是为了让模型能够像工程师一样编写并运行代码，用确定性计算替代“脑算”。它属于受管工具（managed tool），执行发生在 Anthropic 平台端，与 Files API、web search/fetch 深度集成，支撑数据分析、可视化和 Agent 工作流。

## 痛点

没有代码执行时，Claude 只能直接给出计算结果，遇到大数运算、复杂统计或解析 CSV 等内容容易产生幻觉或错误。若采用常规 function calling 让客户端执行代码，开发者必须自行实现沙箱、管理执行环境、处理结果回传和状态维护，复杂度高且安全风险大。

## 解决办法

Claude API 将代码执行做成一个声明式工具，请求中传入 tools 后，模型自己判断何时需要运行代码。API 在服务器端沙箱容器内自动执行模型发起的 Bash 命令和 Python 文件操作，容器无外网、只包含预装库，因此不能下载运行时依赖，但能保证安全隔离。每次执行的结果会作为 tool_result 自动回到模型上下文，模型看到真实输出后继续推理或修正代码。容器 ID 允许在后续请求中复用同一个环境，20260120 及以上版本还支持 REPL 状态持久化，让变量和文件跨请求保留。这相当于给模型配了一个一次性、隔离的云端终端：它写代码、运行、看结果，与人类工程师的调试循环一致。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=2048,
    tools=[{'type': 'code_execution_20250825', 'name': 'code_execution'}],
    messages=[{'role': 'user', 'content': '用 Python 计算 2**100 并写入 result.txt'}],
)
print(resp.container.id)  # 后续请求可传 container_id 复用沙箱
for block in resp.content:
    if block.type == 'server_tool_use':
        print('Claude 调用:', block.name, block.input)
    elif block.type == 'text':
        print('Claude 回复:', block.text)
```

这段代码创建 API 请求并声明 code_execution_20250825 工具。模型收到用户消息后，会返回 server_tool_use 块表示它要执行 Bash/文件操作；API 在服务器端沙箱自动执行并把结果作为 tool_result 插回响应，因此客户端无需自行执行或回传结果。响应的 container.id 用于后续复用同一容器，是状态持久化的关键。

## 关键流程

1. 在 API 请求的 tools 数组中声明 code_execution 工具（选择合适版本）。
2. 模型评估请求是否需要代码执行，若需要则生成 Bash 命令或文件操作。
3. API 在无网络的沙箱容器中执行这些操作，并自动把结果返回给模型。
4. 模型读取执行结果，继续推理、修正或生成最终回答/文件。
5. 如需处理用户上传文件，先用 Files API 上传并通过 container_upload 内容块引用。
6. 如需跨请求保持状态，传递之前响应中的 container.id 复用容器。

## 关键点

- 代码执行工具同时提供 Bash 命令和文件操作能力，使 Claude 能完成真实计算、数据解析、可视化和文件生成，而不只是生成建议代码。
- 所有代码在 Anthropic 托管的无网络沙箱中运行，避免了模型执行任意代码时攻击宿主或泄露数据的风险，但代价是不能安装运行时依赖。
- API 会自动执行命令并回传结果，客户端不必实现工具执行逻辑或把 tool_result 发回，这让 Agent 循环更简洁。
- 容器 ID 复用是跨请求状态管理的基础，配合 20260120+ 的 REPL 持久化可以实现长程迭代式分析。
- 版本选择应考虑模型兼容性和功能需求：20250825 足够做 Bash 和文件操作，20260120+ 适合需要程序化工具调用或持久化的场景，20260521 还帮助模型管理 90 秒 cell 超时。
- 与 web search/fetch 配合时，代码执行可用于动态过滤搜索结果，且不额外收费，这是降低上下文噪声的重要机制。

## 对比与权衡

- 相比常规 function calling，这个方案在无需自建执行环境、自动安全隔离和状态托管上更好，但在执行环境定制、依赖安装和工具语义灵活度上不如常规 function calling。
- 相比在本地自建 Docker 沙箱执行代码，这个方案在开箱即用、免运维和统一安全策略上更好，但在网络访问、自定义依赖和持久化资源控制上不如本地容器。
- 相比 OpenAI 的 Code Interpreter，两者定位相似；Claude 的代码执行与 web_search/web_fetch 动态过滤集成更紧密，但在生态成熟度和部分企业功能上可能不如 OpenAI。

## 自测问题

**问: 代码执行工具和普通 tool use / function calling 有什么区别？**

普通工具调用由客户端执行函数并回传 tool_result；代码执行由 API 服务器端沙箱托管执行，客户端只声明工具并观察结果。优点是安全、省去执行环境搭建，缺点是无法自定义依赖和网络行为。

**问: 为什么沙箱没有互联网访问？如果我想安装 pandas 之外的库怎么办？**

无网络是为了防止模型运行任意代码时访问外网、攻击内网或泄露数据。依赖只能使用预装库，当前不能运行时 pip install；需要额外库时应考虑自定义工具或在本地执行方案中实现。

**问: container.id 复用的机制是什么？为什么 20260120 之前复用容器也不能持久化 Python 变量？**

容器复用保持文件系统和进程环境存活，所以文件能留下来。但 Python 解释器状态持久化需要额外保存 REPL 会话，20260120 才引入该能力；旧版本即使复用容器，变量绑定也不跨请求保留。

**问: 如果模型运行一个耗时过长的命令会怎样？**

整个工具调用超过最大执行时间会返回 execution_time_exceeded 错误。20260521 在程序化工具调用中对每个 Python cell 有 90 秒墙钟限制，超时返回非零 return_code 和 detection_timeout，模型可据此调整代码。

**问: 在 Agent 场景中，代码执行工具如何与文件上传和生成文件配合？**

用户通过 Files API 上传 CSV/图片，消息中用 container_upload 引用，模型在沙箱里用 Python 读取并分析；生成的文件写入输出目录后通过响应中的文件内容返回，或继续在同一容器中供后续步骤使用。

## 适用场景

- 数据分析与可视化：上传 CSV、Excel 或图片，让 Claude 使用 pandas/matplotlib 进行探索性分析并生成图表。
- 精确数值计算与验证：大整数运算、复杂公式推导、模拟仿真等需要确定性结果的场景。
- 文件生成与格式转换：让 Claude 生成文本、代码、图像等文件，并在输出目录中捕获下载。
- 多轮 Agent 工作流：结合容器复用和程序化工具调用，让 Claude 在多次请求中迭代处理同一批数据或维护状态。

## 标签

`Claude API` `Tool Use` `代码执行` `沙箱` `Agent`

# Claude API Agent Skills 入门：用容器技能生成 PPT/Excel/Word/PDF

*原文: [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart) · 来源: web · 生成时间: 2026-09-20T06:16:46.152697+00:00*

## 背景

大模型通常以文本输出为主，直接生成 Office 或 PDF 二进制文件容易出错、格式不可控。Agent Skills 把某类任务的专家指令、脚本、依赖和运行方式封装成可复用单元，让 Claude 在受控容器中调用工具生成符合格式的文件。这样既降低开发者编写复杂 prompt 的成本，也避免每个请求都携带大量样板指令。

## 痛点

没有 Agent Skills 时，开发者要让模型生成 pptx/xlsx 往往需要自己准备 XML 结构、渲染脚本或 base64 模板，提示词冗长且输出不稳定；同时如果一次性加载多个技能说明，会挤占上下文窗口并增加延迟。

## 解决办法

请求通过 container.skills 声明允许使用的技能列表，并开启 code_execution_tool。Claude 启动时只读取技能的 name/description 元数据，这是第一级渐进披露；收到任务后根据语义匹配到对应技能，再加载完整指令和脚本，这是第二级渐进披露。技能在代码执行容器内运行并生成文件，响应中返回 file_id，用户通过 Files API 下载结果。可以类比为：先查看工具箱目录判断是否需要某个工具，确定后再打开该工具的详细说明书，而不是一开始把所有说明书都背在身上。

## 关键代码示例

```python
import anthropic, requests, os

client = anthropic.Anthropic()
resp = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=4096,
    tools=[{'type': 'code_execution_tool'}],
    container={'skills': [{'type': 'anthropic', 'skill_id': 'pptx', 'version': 'latest'}]},
    messages=[{'role': 'user', 'content': 'Create a 5-slide presentation about renewable energy.'}]
)

file_id = next(
    block.file.file_id for block in resp.content
    if getattr(block, 'file', None)
)

r = requests.get(
    f'https://api.anthropic.com/v1/files/{file_id}/content',
    headers={'x-api-key': os.environ['ANTHROPIC_API_KEY']}
)
open('renewable-energy.pptx', 'wb').write(r.content)
```

第一段创建消息时，container.skills 声明了 Anthropic 托管的 pptx 技能，tools 中开启 code_execution_tool，这是技能运行所必需的。模型不会返回 base64 文件，而是在代码执行容器中生成文件，并在响应内容里给出文件引用。第二段从响应中提取 file_id，第三段调用 Files API 下载生成的文件字节流。

## 关键流程

1. 调用 Skills API 或列出可用技能，确认 pptx、xlsx、docx、pdf 等技能 ID。
2. 在 Messages API 请求中通过 container.skills 指定 Anthropic 技能、skill_id 和 version，并启用 code execution tool。
3. 用户消息描述任务；Claude 根据语义自动匹配技能并执行生成文件。
4. 从响应中提取文件引用 file_id。
5. 通过 Files API 下载文件到本地。

## 关键点

- container.skills 是声明可用技能的入口，不声明则 Claude 不会使用该技能。
- 渐进披露分两级：启动时只加载技能元数据，匹配任务后再加载完整指令，显著节省上下文窗口。
- Skills 依赖 code execution tool 才能实际生成二进制 Office 或 PDF 文件，因此必须显式开启 tools。
- 生成结果不是直接返回 base64，而是通过文件引用 file_id 加 Files API 下载，这更适合大文件传输。
- version 使用 latest 可以跟随最新发布，但生产环境建议锁定具体版本以保证可复现性。

## 对比与权衡

- 相比直接在 prompt 中要求模型输出 Office XML 或 Markdown 后再用 pandoc/libreoffice 转换，Agent Skills 封装了经过调优的生成逻辑与依赖，格式更稳定，且不占用用户上下文。
- 相比通用 function calling 或 MCP 自定义工具，Agent Skills 是 Anthropic 托管的垂直技能，接入更快、免运维，但在连接私有数据源、企业系统或任意外部 API 上不如 MCP 或自定义 tool 灵活。

## 自测问题

**问: Agent Skills 为什么要做两级渐进披露？**

如果启动就加载所有技能完整说明，上下文会膨胀，增加成本和延迟，也容易干扰模型决策；先加载 name/description 让模型知道有哪些技能，等确定任务相关后再加载详细指令，类似工具选择中的按需加载。

**问: 为什么需要 code execution tool？**

Skills 需要运行脚本来操作 pptx、xlsx、docx、pdf 等二进制结构，模型本身不会直接在推理中生成文件；code execution 提供隔离环境执行技能代码，生成文件保存在容器中，再通过文件引用返回。

**问: 多个 skills 同时传入时 Claude 怎么选择？**

模型依据用户请求与每个技能 description 的语义相关性进行匹配；通常会自动选择最匹配的 skill，也可以在 prompt 中明确指定要使用的技能来约束。

**问: 如何保证生成文件可复现和生产可控？**

可以通过 version 锁定技能版本而不是 latest；把 skill 视为依赖管理，记录输入 prompt 和版本，必要时使用低 temperature 或显式约束内容结构，并进行自动化校验。

**问: Agent Skills 与 MCP 或 function calling 有什么区别？**

Skills 封装了“怎么做某类任务”的流程知识和执行代码，适合标准化文件或数据处理；MCP 和 function calling 更偏向“连接外部能力”，适合动态查询、自定义 API 和内部系统。两者可以组合使用。

## 适用场景

- 需要批量或自动化生成格式规范的 PPT、Excel、Word 或 PDF 报告。
- 把自然语言需求转成可下载的办公文档，例如周报、商务方案或合同草稿。
- 在 RAG 或 Agent 流程中作为输出组件，把分析结果写成 Excel 或 PDF 交付。
- 快速验证 Anthropic Agent Skills 能力，再决定是否自建垂直技能。

## 标签

`Agent Skills` `Claude API` `文档生成` `渐进披露` `Code Execution`

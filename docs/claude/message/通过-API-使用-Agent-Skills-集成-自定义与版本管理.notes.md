# 通过 API 使用 Agent Skills：集成、自定义与版本管理

*原文: [https://platform.claude.com/docs/en/build-with-claude/skills-guide](https://platform.claude.com/docs/en/build-with-claude/skills-guide) · 来源: web · 生成时间: 2026-09-20T03:18:34.170166+00:00*

## 背景

Agent Skills 是为了把“让 Claude 处理文件、运行脚本”这类复杂能力标准化而出现的。此前开发者需要在提示词里反复描述操作步骤，或自行拼装 tools 和代码执行逻辑，难以复用和版本化。Skills 把这些指令、脚本和资源打包成可发现、可挂载的单元，并通过代码执行容器安全运行。对 API 用户来说，这意味着既能使用 Anthropic 预置的 Office/PDF 技能，也能上传私有技能。

## 痛点

没有 Skills 时，处理 Excel/PPT/PDF 或私有脚本任务往往需要手写 tool schema、提示词和事后处理，容易出错且每次请求都要重复。如果不懂 container 挂载方式，就无法在 API 中启用这些技能，只能退回到模型直接生成内容，质量不稳定且不能执行真实文件操作。

## 解决办法

核心是把技能作为容器内的资源包挂载：请求中通过 container 参数指定 type、skill_id 和可选 version，同时必须启用 code_execution_tool。Claude 在系统提示中看到技能的名称和描述，按需把文件复制到 /skills/{skill-name}/，并自动调用脚本执行。生成的文件通过响应里的 file_id 返回，需要再调用 Files API 下载；多轮任务复用容器 ID，长任务处理 pause_turn。可以类比为给 Claude 装上即插即用的工具箱：每个箱子自带说明书和可执行脚本，而不是每次临时口头教它怎么做。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

# 1. 启用代码执行工具，并通过 container 挂载自定义 Skill
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=4096,
    tools=[{"type": "code_execution_tool"}],
    container={
        "type": "custom",
        "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
        "version": "latest",
    },
    messages=[{"role": "user", "content": "用我的财务技能生成一份季度损益表 Excel"}],
)

# 2. 从响应中的代码执行工具结果块提取 file_id
file_id = None
for block in response.content:
    if block.type == "tool_use" and block.name == "code_execution_tool":
        file_id = block.input.get("file_id")
        break

# 3. 使用 Files API 下载生成的文件
if file_id:
    file_bytes = client.files.download(file_id)  # 具体方法以官方 SDK 为准
    with open("output.xlsx", "wb") as f:
        f.write(file_bytes)
```

这段代码展示了 Skill 集成的三个关键步骤：创建消息时必须在 tools 中启用 code_execution_tool，同时 container 参数告诉 API 要挂载哪个技能及版本；响应中生成的文件不会直接返回二进制，而是通过 file_id 暴露在代码执行工具的结果块里；最后用 Files API 下载。用 Anthropic 预置技能只需把 type 改为 "anthropic"、skill_id 改为 "xlsx" 等短名，其余结构完全一致。

## 关键流程

1. 启用代码执行工具，并选择兼容模型（见 code execution tool 兼容列表）。
2. 在 Messages API 请求中通过 container 参数指定技能的 type、skill_id 和可选 version，单次最多 20 个。
3. 若技能生成文件，从响应中的 code-execution 工具结果块提取 file_id，再调用 Files API 下载。
4. 多轮对话复用响应 container 对象中的 container id；长任务根据 stop_reason 为 pause_turn 继续交互。
5. 自定义技能：创建含 SKILL.md 的目录，打包上传（zip 或文件对象），创建版本并指定 latest 或固定 skver ID。

## 关键点

- Anthropic 预置 Skills 与自定义 Skills 的集成结构完全一致，只是 type 和 skill_id 来源不同；因此切换或组合两者不需要改代码。
- container 参数是 Messages API 中挂载 Skills 的唯一入口，且必须与 code_execution_tool 一起使用；把 Skill 当成独立工具会导致请求失败。
- 自定义 Skill 必须包含 SKILL.md，其 YAML frontmatter 对 name 和 description 有严格约束；这既保证模型能准确发现技能，也防范提示注入和路径冲突。
- 版本管理上，Anthropic 技能用日期版本，自定义技能用 skver_ ID；生产环境应固定版本而不是 latest，因为新版本是完整快照，可能引入不可预期变化。
- 生成文件通过 file_id 返回并需用 Files API 下载；容器可以跨多轮复用，让长任务保持同一文件系统状态。

## 对比与权衡

- 相比直接使用 code execution tool 并手写提示词/脚本，Agent Skills 把指令、脚本和资源打包成可复用单元，在多轮和重复任务上更稳定、容易维护，但需要提前上传和版本管理，灵活性略低。
- 相比 Anthropic 预置 Skills，自定义 Skills 能封装私有工作流和业务脚本，但需要自己管理上传、权限和版本，且无法享受 Anthropic 的更新维护。

## 自测问题

**问: Agent Skills 和普通 function calling / tool use 有什么区别？**

Skills 不是新的工具类型，而是通过 code execution tool 挂载的资源包。它提供的是可自动加载的指令、脚本和资源，模型按需调用，不需要开发者手动定义 function schema；它更适合多步文件处理和代码执行，而 function calling 更偏结构化 API 调用。

**问: 为什么使用 Skills 必须启用 code execution tool？**

Skills 的脚本需要在隔离容器中运行，code execution tool 提供执行环境、文件系统和容器生命周期。没有它，Skill 文件无法被复制和执行，所以请求中必须同时声明 tools 中的 code_execution_tool，并使用兼容模型。

**问: 自定义 Skill 的 SKILL.md 有哪些硬性约束？为什么这么设计？**

name 最长 64 字符，只能小写字母/数字/连字符，不能包含 XML 标签和保留词；description 最长 1024 字符、非空且无 XML 标签。这些限制是为了防止提示注入、避免文件路径冲突，并保证技能在系统提示中能安全、无歧义地被模型发现。

**问: 版本管理中固定版本和 latest 如何取舍？**

开发调试阶段可以用 latest 快速获得最新能力；生产环境应固定日期版本（Anthropic）或 skver ID（自定义）。因为版本是完整快照而非增量，升级可能改变行为，固定版本可保证请求结果可复现、可回滚。

**问: 生成的文件如何从响应中下载？**

Skills 在代码执行过程中生成文件，响应中的 code-execution 工具结果块会包含 file_id。开发者必须先提取 file_id，然后调用 Files API 下载实际内容；Messages API 响应不会直接返回二进制文件。

## 适用场景

- 批量生成 Office 文档：使用 Anthropic 预置 xlsx、pptx、docx 技能自动生成报表、演示文稿和 Word 文档。
- 私有数据处理流水线：上传自定义 Skill，在容器内运行自己的脚本处理上传的 CSV/PDF，输出结构化结果。
- 多技能组合工作流：例如先用 PDF 技能解析文档，再用 xlsx 技能汇总表格，单请求挂载多个技能。
- 需要隔离执行环境的自动化任务：通过代码执行容器运行不受本地依赖影响的脚本。

## 标签

`Agent Skills` `Claude API` `code execution` `文件生成` `技能管理`

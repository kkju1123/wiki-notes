# Claude Memory Tool：跨会话记忆与按需检索

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) · 来源: web · 生成时间: 2026-09-20T03:56:32.423065+00:00*

## 背景

长期运行的 AI Agent 需要跨会话保留上下文，但上下文窗口有限且成本高。Memory tool 把记忆外置为可持久化文件，由模型在任务前按需读取，避免把所有历史都塞进 prompt。Anthropic 将其作为内置工具提供，存储仍由应用方控制。

## 痛点

没有记忆时，每次会话都要重新介绍背景，多会话项目无法延续。若把所有历史全量加载，会迅速耗尽上下文窗口、引入噪音并增加 token 成本。自己实现记忆又容易忽略路径安全，造成任意文件读写漏洞。

## 解决办法

Memory tool 定义了一组文件操作命令（view/create/update/delete 等），Claude 在任务开始或需要时通过 tool_use 请求这些操作。客户端 handler 把虚拟路径 /memories/* 映射到自己的存储（文件、数据库等），执行后以 tool_result 返回文本。模型只读取当前任务需要的记忆文件，实现即时上下文检索；同时所有路径都被强制限制在 /memories 前缀内，防止目录穿越。类比：像给模型一个专属笔记本，它只在需要时翻看相关页，而不是背下整本。

## 关键代码示例

```python
import os
from pathlib import Path

MEMORY_ROOT = Path("./memory_store").resolve()

def handle_memory_command(input_data: dict) -> str:
    command = input_data["command"]
    path = input_data["path"]  # 例如 "/memories/project/notes.txt"

    # 1. 必须限制在 /memories 前缀内
    if not path.startswith("/memories"):
        return "Invalid path: must be under /memories"

    rel = path.removeprefix("/memories").lstrip("/")
    full = (MEMORY_ROOT / rel).resolve()

    # 2. 防目录穿越：解析后仍必须在根目录内
    if not str(full).startswith(str(MEMORY_ROOT)):
        return "Invalid path: path traversal detected"

    if command == "view":
        if full.is_dir():
            return format_directory_listing(full)
        return format_file_with_line_numbers(full)

    if command == "create":
        if full.exists():
            return f"File already exists: {path}"
        full.parent.mkdir(parents=True, exist_ok=True)
        full.write_text("")
        return f"File created successfully at: {path}"

    # update / delete 等其他命令按同模式分发
    return f"Unknown command: {command}"
```

这段代码展示 handler 的核心：先把模型传入的 /memories 虚拟路径转换为真实存储路径；通过前缀校验和 resolve 后仍位于根目录双重检查防止 ../ 目录穿越；然后按 command 分发到 view/create 等操作，返回字符串给 tool_result。实际生产还要处理行号范围、图片、行数上限等细节。

## 关键流程

1. 在请求 tools 中加入 {"type":"memory_20250818","name":"memory"}，不需要自定义 input schema。
2. 实现客户端 handler，接收 tool_use 的 command/path 等输入并执行真实存储操作。
3. 对所有路径强制校验 /memories 前缀，并将其映射到受控根目录后防穿越。
4. 按命令返回规范字符串（或被模型读取的自定义文本）作为 tool_result，继续工具循环。

## 关键点

- Memory tool 是 Anthropic 提供的内置工具接口，Claude 只发 tool_use 请求，存储由应用方完全控制，适合私有部署和数据合规。
- 所有文件操作必须被限制在 /memories 目录内，并防止绝对路径、..、符号链接等方式逃逸，否则有任意文件读写风险。
- 该工具遵循标准 tool-use 循环：模型请求 -> 应用执行 -> tool_result 回传 -> 模型继续生成，这是与外部系统交互的基础。
- 模型会主动在任务前检查记忆目录并按需读取，而不是全量预加载，这能保持上下文聚焦并降低成本。
- view 命令对目录和文件都有明确的返回格式（如行号右对齐、Tab 分隔、空目录不是错误），这些约定降低模型解析错误率。

## 对比与权衡

- 相比全量加载历史上下文，memory tool 的按需检索在 token 消耗和上下文聚焦上更好，但依赖模型判断何时读取，可能遗漏未被主动检索的相关记忆。
- 相比外部向量数据库/RAG 的语义检索，文件系统式 memory 更透明、可人工审计和编辑，但在按内容语义召回上不如向量检索。
- 相比服务端托管长期记忆，客户端 handler 让数据留在应用基础设施内、隐私可控，但需要开发方自行处理存储可靠性、多租户隔离和并发安全。

## 自测问题

**问: 为什么 memory 要设计成 client-side，而不是 Anthropic 服务端帮忙存？**

核心是数据控制权：企业或用户往往要求记忆数据留在自己的 VPC、数据库或加密文件里；客户端模式让存储介质可替换，Anthropic 只定义工具协议，避免了将敏感上下文传到第三方。实现上必须自己处理安全、并发和持久化。

**问: 如何防止 path traversal 攻击？**

首先校验传入 path 是否以 /memories 开头；然后拼接真实根目录并使用 resolve() 规范化，再确认规范化后的路径仍在根目录内；同时拒绝符号链接、绝对路径和 ..，最好在 OS 级限制 handler 只能访问该目录。

**问: view 命令对空目录应该返回什么，为什么重要？**

空 /memories 目录不是错误，应返回列表头加目录自身的大小和路径行。如果返回错误，模型可能误以为记忆系统不可用并停止尝试读取，破坏自动检索流程。

**问: 什么时候应该用 memory tool，而不是直接扩大上下文窗口？**

当会话跨越很长时间、任务之间只有部分信息相关、需要积累跨用户知识时用 memory；如果任务本身很短或信息量小，直接放上下文更简单。另一个判断标准是数据是否需要持久化和审计，需要就外置记忆。

**问: 实现生产级 memory handler 除了 CRUD 还需要考虑什么？**

行数/文件大小限制、图片文件返回、并发写冲突、幂等 create、多租户路径隔离、加密、备份和审计日志；还需要避免隐藏文件被模型读出，以及控制目录列表深度。

## 适用场景

- 跨多个 agent 会话维护长期项目上下文，例如持续开发同一代码库。
- 构建个人/团队知识库，把每次交互的决策、反馈和经验沉淀为可复用文件。
- 客服或支持系统在多轮工单间记住用户背景和偏好。
- 需要审计或人工编辑记忆内容的场景，文件系统式存储方便直接查看和修改。

## 标签

`Memory Tool` `Context Engineering` `Tool Use` `Claude` `Agent Memory`

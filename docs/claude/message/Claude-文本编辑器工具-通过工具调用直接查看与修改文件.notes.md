# Claude 文本编辑器工具：通过工具调用直接查看与修改文件

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool) · 来源: web · 生成时间: 2026-09-20T03:59:52.792235+00:00*

## 背景

LLM 早期只能生成修改建议，用户需要手动复制代码和文本，容易出错且无法形成“读取—修改—验证”的闭环。Anthropic 的 text editor tool 把文件系统操作包装成模型可调用的内置工具，让 Claude 在调试、重构、测试生成等 agentic 场景中直接操作文件。

## 痛点

没有这个工具时，模型看不到真实文件内容，只能靠用户粘贴片段，容易因上下文不完整误判问题；即使用户复制建议，也无法自动应用修改，迭代成本高。

## 解决办法

工具采用 schema-less 设计，schema 内置于 Claude，无需应用侧定义 JSON schema。它提供 view（按行范围读取/列目录）、str_replace（用 old_str 精确匹配并替换为新字符串）、create（新建文件）和 insert（在指定行后插入）四类命令。应用侧在收到 tool_use 后执行真实文件操作，并把结果包装为 tool_result 返回，模型据此继续判断或输出最终解释。可以把它类比为给模型一个受控的“文件编辑权限”，而不是无约束的 shell 访问。执行时必须做路径校验、备份和错误处理，因为 str_replace 的 old_str 必须完全匹配，包括空白和缩进。

## 关键代码示例

```python
from pathlib import Path
import shutil

ROOT = Path('/workspace')

def handle(t):
    cmd = t['command']
    p = (ROOT / t['path']).resolve()
    if ROOT.resolve() not in p.parents and p != ROOT.resolve():
        raise ValueError('path outside root')

    if cmd == 'view':
        lines = p.read_text().splitlines()
        s, e = t.get('view_range', [1, -1])
        e = len(lines) if e == -1 else e
        return {'content': chr(10).join(lines[s - 1:e])}
    if cmd == 'str_replace':
        text = p.read_text()
        if t['old_str'] not in text:
            raise ValueError('old_str not found')
        shutil.copy(p, str(p) + '.bak')
        p.write_text(text.replace(t['old_str'], t['new_str'], 1))
        return {'status': 'ok'}
    if cmd == 'create':
        p.write_text(t['file_text'])
        return {'status': 'created'}
    if cmd == 'insert':
        lines = p.read_text().splitlines()
        lines.insert(t['insert_line'], t['insert_text'])
        p.write_text(chr(10).join(lines))
        return {'status': 'inserted'}
    raise ValueError('unknown command')

```

这段代码展示应用侧如何处理 Claude 发来的 text editor tool 调用：先 resolve 路径并校验必须在 ROOT 内，防止目录穿越；然后按 command 分支执行 view/str_replace/create/insert。str_replace 只做精确匹配，用 count=1 替换第一个匹配，并在改前备份为 .bak；insert 用 lines.insert 实现“在第 N 行后插入”，0 表示文件开头。

## 关键流程

1. 在 API 请求中配置 text editor tool，并给 Claude 一个需要查看或修改文件的提示。
2. Claude 先调用 view 命令读取文件内容或列目录。
3. 应用侧执行 view，读取文件/目录并按 max_characters 或 view_range 截断，将结果作为 tool_result 返回。
4. Claude 根据内容决定修改方式，调用 str_replace、insert 或 create。
5. 应用侧执行修改命令，完成替换/插入/创建，并把结果返回给 Claude。
6. Claude 综合前后信息，给出问题分析和修改说明。

## 关键点

- text editor tool 是 schema-less 工具，input schema 内置于 Claude，无法修改；这降低了配置复杂度，但也意味着命令集是由 Anthropic 固定的。
- view 支持行号范围和 max_characters 截断，模型可以按需读取大文件局部，而不是把整个文件塞入上下文。
- str_replace 的 old_str 必须精确匹配包括空白和缩进；实现方不应做模糊匹配，而应返回错误让模型重新 view。
- insert 的行号是“在该行之后插入”，0 表示文件开头，处理时必须注意这个 1-indexed 语义。
- 实现方必须具备路径校验、备份和错误处理，因为模型写文件是真实副作用操作。
- 工具调用有额外输入 token 成本（例如 Claude 4 的版本为 700 tokens），多工具组合时需计入总成本。

## 对比与权衡

- 相比用户手动复制粘贴代码给模型，text editor tool 能直接读写文件，减少信息丢失和人工往返，但对应用侧的文件系统权限和安全控制要求更高。
- 相比通用 function calling 自定义工具，text editor tool 无需定义 JSON schema 且模型已针对命令进行优化，但灵活性差，不能新增或修改命令参数。
- 相比旧版 text_editor_20250124，新版 20250429 移除了 undo_edit 命令，20250728 增加 max_characters 参数；丢失内建撤销意味着应用侧必须依赖备份或版本控制。

## 自测问题

**问: text editor tool 和普通 function calling 工具有什么区别？**

它是 schema-less 的内置工具，schema 在模型内部，配置时不能自定义参数；而普通 function calling 需要应用提供 JSON schema。它专门为文件编辑设计，命令集固定为 view/str_replace/create/insert，模型对这些命令的理解较稳定。实现时仍需应用执行真实文件操作并返回 tool_result。

**问: 为什么 str_replace 用 old_str 精确匹配，而不是行号或正则替换？**

精确匹配能利用模型对文件内容的观察，避免行号变化导致误改；相比正则，它更可读、可验证，且对大模型生成友好。但代价是必须逐字符匹配空白/缩进，若文件已经变化就会失败，因此实现方应返回错误让模型重新查看。行号插入仍用于 insert 命令，因为插入无需删除旧文本。

**问: 如何防止 Claude 使用该工具破坏文件？**

应用侧需要限制可访问根目录，resolve 后校验路径前缀防止 ../ 穿越；在写操作前自动备份，如 .bak 或 git commit；对敏感路径做权限检查；并且把工具限制在用户明确指定的工作区。模型侧可以通过 view_range 降低误读范围，但最终安全边界在应用。

**问: 查看大文件时如何控制 token 消耗？**

view 命令支持 view_range 只读指定行区间，工具配置支持 max_characters 截断；通常模型会分多次局部读取，而不是一次全量读入。应用侧还可以返回目录列表让模型先定位文件。若文件特别大，可配合搜索工具或先读取关键区域。

**问: 为什么要实现成无 schema 工具？这样不是少了校验吗？**

schema-less 工具将命令协议固化到模型训练中，减少运行时 JSON schema 解析和错误；适合命令固定且高频的文本编辑场景。副作用是应用不能依赖 schema 校验，需要自己在 handler 中做参数类型/范围检查，例如 insert_line 必须是 int 等。

## 适用场景

- 代码调试：模型直接读取报错文件和上下文，定位语法/逻辑错误并给出补丁。
- 批量重构：在多个文件中做函数重命名、抽取等精准修改，减少手动复制。
- 文档与注释生成：让 Claude 为代码库补充 README、docstring 或注释。
- 测试生成：分析实现后直接创建测试文件，或向已有测试插入用例。

## 标签

`Claude` `工具使用` `文本编辑器` `Agent` `文件操作`

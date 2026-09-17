# Karpathy 式知识栈：Hermes + MiniMax M3 + Obsidian 如何组成第二大脑

*原文: [https://x.com/polydao/status/2066904909849440434](https://x.com/polydao/status/2066904909849440434) · 来源: twitter · 生成时间: 2026-09-17T03:25:37.541392+00:00*

## 背景

PKM 从文件夹式笔记演进到 Obsidian/Roam 的双链图谱，但把 LLM 接入知识库时，多数做法仍停留在聊天问答，导致会话失忆；普通 RAG 切块又会丢掉链接与分类法。这套组合要解决的是让 AI 在完整知识图谱内持续工作，而不是每次只读取三张卡片。

## 痛点

没有这种架构时，笔记写完就沉底，AI 会话每次从零开始；一跨十几个笔记或多个 MOC，模型常在十几次工具调用后开始 drift，局部通顺但全局错误，甚至会引用不存在的标签和 wikilink。

## 解决办法

把 Obsidian 当唯一 ground truth：所有值得保留的内容先落成本地 markdown，并用双链和固定 taxonomy 表达关系。Hermes 作为编排层读取 vault、调用工具、记住历史并写回结果，不替代 Obsidian。MiniMax M3 则一次性把整张图谱和 taxonomy 放入长上下文，而不是只喂检索切块。类比：像给卡片盒配了一个能同时看到所有卡片、做完还会归档整理的常驻实习生，最终形成 capture → read → reason → write-back 的闭环。

## 关键代码示例

```python
import pathlib, openai

vault = pathlib.Path('./vault')
ctx = '\n\n'.join(p.read_text() for p in vault.rglob('*.md'))
tags = sorted({l.strip() for p in vault.rglob('*.md') for l in p.read_text().splitlines() if l.startswith('#')})

client = openai.OpenAI(base_url='https://api.minimax.io/v1', api_key='YOUR_KEY')
resp = client.chat.completions.create(
    model='MiniMax-M3',
    messages=[{'role': 'user', 'content': (
        'Use existing taxonomy and [[wiki links]]; write [[forward-ref]] if missing.\n\n'
        f'VAULT:\n{ctx}\n\nTAXONOMY:\n{tags}')}],
    temperature=0.2,
)
pathlib.Path('./vault/_inbox/synthesis.md').write_text(resp.choices[0].message.content)
```

这段代码不做向量检索，而是把所有 markdown 拼成一个 context，让模型同时看到整张图；提示词强制沿用已有 taxonomy，并允许不存在的页写成 [[forward-ref]]。最后写回 `_inbox/synthesis.md`，对应 Hermes 将推理结果沉淀为 Obsidian 笔记的闭环。

## 关键流程

1. 在本地建立 Obsidian vault，并定义固定 taxonomy，例如 #coin/*、#project/*、#concept/*、#meta。
2. 安装 Hermes Agent；桌面端用于日常交互，CLI 用于脚本、远程和可复现部署。
3. 将 vault 目录路径通过配置或环境变量暴露给 Hermes，确保 agent 可以直接读写 markdown。
4. 在 Hermes 中把 MiniMax M3 设为默认模型，让它在任务中读取整个 vault 或相关 MOC。
5. 定义循环任务：捕获原始笔记 → Hermes 读取相关图 → M3 重构、打标签、链接 → 写回 Obsidian。
6. 每周跑 vault lint，处理 forward references、重复 tag、失效 wikilink。

## 关键点

- Obsidian 作为 ground truth，把知识表达成可遍历的本地 markdown 图和双链；这是 agent 可维护性的基础。
- Hermes 不是套壳聊天，而是有持久记忆和工具调用能力的编排层，位于 vault 与模型之间。
- MiniMax M3 的选择标准不是 benchmark，而是能否在整张 vault 图进入 context 后仍保持 taxonomy 和任务主线。
- 工具调用越多，一般模型 drift 越严重；M3 在 30+ 次调用下仍一致，因此减少手动清空或重建会话的频率。
- 让模型写 [[forward-ref]] 是保留思考结构的刻意设计，但必须用每周 lint 兜底。

## 对比与权衡

- 相比普通 RAG 切块检索，这个方案保留双链、MOC 和 taxonomy 的全局结构，但长上下文成本和延迟更高，vault 超大时仍需 agent 做过滤或分层。
- 相比 Notion 或云笔记里的 AI 问答，本地 markdown + Hermes 更可版本化、可脚本化且避免供应商锁定，但需要自己维护 agent 环境。
- 相比只用 200K 上下文模型处理 vault，M3 在长 agentic loop 和多标签一致性上更稳；但首字延迟较高，重图 PDF 的视觉理解不如专用视觉工具。
- 相比自训练或微调个人模型，调用长上下文 API 模型无需训练基础设施，但依赖外部 API 并有运行成本。

## 自测问题

**问: 为什么要把整个 vault 放进 context，而不是做 RAG？**

RAG 的检索切块会破坏图谱的全局一致性，模型容易只根据片段生成局部通顺、全局错误的结果；长上下文让模型同时看见 taxonomy、MOC 和 wikilink。但纯长上下文不适合超大知识库，实际可先让 agent 缩小候选范围，再用 M3 全量读候选子图。

**问: 什么是 agent 在长任务里 drift？怎么解决？**

多轮工具调用中，早期指令被后续结果挤出有效注意力，模型从第 8-9 步开始偏离主任务。解决：选长上下文稳定性强的模型、提供结构化 MOC、把中间状态写回文件、减少 session rotation。

**问: 如何避免模型编造笔记链接？**

把它定义为 forward reference 而不是错误；Prompt 中强制 taxonomy 和已有链接；写回前用文件系统校验；每周 lint 处理灰色链接，把它变成真实笔记或显式 TODO。

**问: 为什么用 Obsidian 而不是数据库或 Notion？**

本地 markdown 是最通用、可 diff、可被任意脚本遍历的接口；数据库查询强，但不适合人类日常记录的双链表达；Notion 有云锁定和 API 限制。这个组合让 agent 直接操作文件系统，便于版本控制。

**问: 如果要落地这套 stack，你会怎么评估模型？**

不要只看 benchmark，要构造 vault 任务：读 N 个笔记、交叉引用 MOC、写回新笔记，统计 tag 正确率、假链接率、30+ 工具调用后的任务一致性，同时测首字延迟和成本。

## 适用场景

- 维护个人技术知识库，定期让 agent 跨 MOC 去重、重组、补全双链。
- 长期阅读和写作项目：把多篇文章编译成结构化笔记，并更新相关索引。
- 个人项目日志或周报：每日往 Obsidian 记录原始想法，由 agent 自动整理成任务和概念簇。
- 对隐私和可移植性要求高的 AI 知识管理，不想把所有笔记放进云知识库。

## 标签

`知识管理` `AI Agent` `长上下文模型` `Obsidian` `Hermes`

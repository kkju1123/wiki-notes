# Agent Skills：Claude 的模块化能力扩展机制

*原文: [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) · 来源: web · 生成时间: 2026-09-20T06:14:41.791895+00:00*

## 背景

在大模型 Agent 应用里，领域专家知识通常需要跨会话复用，但仅靠 system prompt 或每轮粘贴长指令会迅速耗尽上下文窗口，且难以维护。Agent Skills 的出现是为了把标准操作流程、参考材料和可执行脚本组织成文件系统目录，让模型像新员工按需查阅入职手册一样工作。这样既不需要重新训练模型，也能把通用 Claude 快速定制成不同领域的专家。

## 痛点

如果把所有领域知识塞进 system prompt，会导致上下文成本高、模型注意力分散，每次切换任务还要重写提示词。让模型临时生成代码执行确定性操作也不稳定且消耗 token。没有 Skills 时，跨会话复用工作流只能靠复制粘贴，容易出错、版本混乱。

## 解决办法

Skills 的核心是文件系统目录 + 三级渐进披露。每个 Skill 在 SKILL.md 的 YAML frontmatter 中声明 name 和 description，这些元数据常驻系统提示（约 100 token），用于触发匹配。当用户请求命中 description 时，Claude 通过 bash 读取 SKILL.md 正文，把 Level 2 指令加载进上下文；如果指令引用其他 markdown、schema 或脚本，则按需读取或执行，脚本源码不进入上下文，只有输出计入 token。这种架构类似图书馆目录：先看卡片（元数据），再翻书（指令），最后查附录或跑工具（资源/代码）。

## 关键代码示例

```markdown
# 目录结构（部署后常驻文件系统，不进入上下文）
pdf-processing/
├── SKILL.md
├── FORMS.md
└── scripts/
    └── extract_text.py

# SKILL.md：level 1 metadata + level 2 instructions
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing
Run `scripts/extract_text.py --input $INPUT` to extract text.
# 触发后 Claude 执行：cat pdf-processing/SKILL.md
# 此时正文进入上下文；只有需要填表时才读 FORMS.md；
# 运行脚本时只有 stdout/stderr 返回，脚本源码不占 token。
```

示例展示一个 pdf-processing Skill 的目录结构和 SKILL.md 内容：frontmatter 的 name/description 是 Level 1，Claude 启动时就加载；正文是 Level 2，只有请求匹配后才通过 bash cat 读取；FORMS.md 和脚本是 Level 3，按需访问。脚本执行时只把输出返回给模型，源码本身不进入上下文。

## 关键流程

1. 创建 Skill 目录，命名为人类可读的 id，如 pdf-processing。
2. 在 SKILL.md 开头写 YAML frontmatter，包含 name 和 description；description 必须同时说明“做什么”和“何时使用”。
3. 在 SKILL.md 正文写操作步骤、规则、最佳实践等 Level 2 指令，控制在 5k token 以内。
4. 将补充 markdown、schema、模板或脚本放入同目录，并在指令中明确何时引用它们。
5. 把 Skill 部署到 Claude Code、API、claude.ai 等目标环境；启动时模型只加载元数据。
6. 用户请求命中 description 后，Claude 自动通过 bash 读取 SKILL.md 并执行相关操作。

## 关键点

- description 是 Skill 触发匹配的唯一判断依据，必须同时说清楚 Skill 做什么以及什么时候用；它决定模型会不会在正确场景自动调用该 Skill。
- 渐进披露是 Skill 的核心架构：Level 1 元数据常驻上下文约 100 token，Level 2 指令触发时读取，Level 3 资源/脚本按需访问；这让大量 Skill 并存而不会撑爆上下文。
- 脚本通过 bash 执行，只有输出进入上下文，源码几乎零 token 成本；适合表单校验、格式转换等确定性任务，比让模型临场生成代码更省更稳。
- Skill 是文件系统目录而不是简单 prompt，因此可以像给新人准备的入职手册一样组织：主文档、专项指南、参考材料和可执行工具各司其职。
- Skill 与 prompt 的本质区别是可复用、自动触发、按需加载；同一套领域知识可以跨会话、跨产品重复使用，避免每次粘贴长指令。

## 对比与权衡

- 相比把所有领域知识直接写入 system prompt，Agent Skills 在上下文成本、可维护性和模块化上更好，但它依赖模型根据 description 自动触发，触发准确性不如显式 prompt 可控。
- 相比 function calling/tool 方案，Skill 更偏领域知识与流程封装，脚本只是其中一种资源；它的调用方式是读文件/跑命令，接口约束弱于专有工具，但知识组合和版本管理更灵活。
- 相比微调（fine-tuning），Skill 不需要训练、可即时更新和组合，也不会改变权重；但在需要模型内部一致性极高、无法依赖指令遵循的任务上，微调可能更稳定。

## 自测问题

**问: Agent Skills 和普通 prompt 有什么本质区别？**

从生命周期、触发方式、上下文占用和可执行资源四方面说明。Prompt 是一次性会话指令，通常需要用户显式提供；Skill 是文件系统资源，元数据常驻、正文按需加载、可跨会话复用，还能带脚本和参考文件。关键词：progressive disclosure、filesystem-based。

**问: 为什么很多 Skill 同时安装也不会显著增加上下文开销？**

因为只有 Level 1 的 name 和 description 常驻 system prompt，每个约 100 token；SKILL.md 正文是在匹配后才用 bash cat 读入，未触发的 Skill 正文和资源不占 token。可以类比图书馆：书脊信息一直在眼前，书的内容打开才看。

**问: Skill 里的脚本为什么比让模型直接生成代码更好？**

脚本源码不进入上下文，只有 stdout/stderr 进入；执行结果确定、可测试、可版本管理；避免模型每次重新生成可能出错且消耗更多 token。适用场景包括校验、格式转换、批量操作。模型生成代码的优势是灵活，适合非标准化任务。

**问: 如何设计一个高质量的 SKILL.md description？**

description 要覆盖“做什么”和“何时使用”，示例：`Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.` 写成名词短语+触发场景，避免只写功能不写场景导致漏触发，也避免太宽泛导致误触发。

**问: Agent Skills 与 MCP 工具/function calling 是什么关系？**

Skills 主要封装领域知识、流程和离线资源，通过文件系统访问；function calling 是模型调用外部 API 的接口；MCP 是标准化的工具/数据源连接协议。三者可以协同：Skill 的指令中可以说明何时调用工具，脚本也可以执行确定性操作，MCP 提供外部能力。区别在于 Skill 是知识/流程资产，工具是执行动作。

## 适用场景

- 企业标准化流程：把销售报价、合同审查、客服工单处理等 SOP 打包成 Skill，让 Agent 按公司规范执行。
- 文档处理任务：使用预构建或自定义 Skill 完成 PDF 文本提取、表单填写、Office 文档生成等常见工作。
- 需要确定性脚本的杂活：例如格式校验、数据清洗、批量重命名；用脚本避免模型生成代码的不稳定和 token 浪费。
- 多步骤复杂任务编排：组合多个 Skill 完成“解析 PDF→提取字段→校验→写入数据库”等跨域流程。

## 标签

`Agent Skills` `Claude` `渐进披露` `上下文工程` `智能体架构`

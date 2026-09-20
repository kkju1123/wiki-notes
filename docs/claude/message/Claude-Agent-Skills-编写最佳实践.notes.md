# Claude Agent Skills 编写最佳实践

*原文: [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) · 来源: web · 生成时间: 2026-09-20T06:18:40.619956+00:00*

## 背景

Claude 的 Agent Skills 机制允许将领域知识、标准流程和脚本打包为可发现的能力单元，作为对底层模型的补充。它解决通用模型在特定任务上缺乏组织知识、指令不稳定，以及把所有 prompt 塞进系统提示导致上下文爆炸的问题。技能通过 SKILL.md 暴露元数据与指令，在需要时才加载详情。

## 痛点

如果不会写 Skill，最直接的问题是 description 太含糊，Claude 在成百上千技能中无法选中正确技能；SKILL.md 写得过长则会挤占对话上下文，干扰后续推理。自由度和测试缺失还会导致高风险操作步骤不严谨，或在不同模型上表现不一致。

## 解决办法

核心做法是把 SKILL.md 当作给聪明新员工的简洁上岗目录：只写 Claude 不知道的领域知识和约束，默认它已掌握通用知识。description 必须同时说明“做什么”和“何时用”，因为它是技能发现的唯一入口。具体执行细节按需拆分到 FORMS.md、reference.md 或 scripts/ 中，使常驻上下文最小化。根据任务脆弱性选择自由度：高风险、必须一致的操作给精确脚本；开放式任务只给原则和启发式。最后在 Haiku/Sonnet/Opus 等目标模型上分别验证，调整说明细度。

## 关键代码示例

```yaml
---
name: processing-pdfs
description: Extract text and form fields from PDF files. Use when the user asks to read, parse, or extract data from PDFs, especially forms or scanned documents.
---

# PDF Processing

Use pdfplumber for text extraction. For form filling, read FORMS.md. For API reference, read reference.md.

## Workflow
1. Run scripts/analyze_form.py on the input PDF.
2. Validate extracted fields with scripts/validate.py.
```

这个最小 SKILL.md 的第一段 YAML 元数据中，name 使用小写连字符的动名词形式，便于引用和检索；description 同时覆盖“做什么”和“何时用”，是模型在多技能中选中它的关键。正文只保留最必要的上下文，不重复解释 PDF 或库是什么；详细表单填写和 API 参考通过 FORMS.md、reference.md 按需加载，脚本只执行不进入上下文。

## 关键流程

1. 用 gerund 形式且仅含小写字母、数字和连字符定义 name，例如 processing-pdfs。
2. 写 description 时同时包含技能功能和触发场景，并覆盖用户可能使用的关键术语。
3. 保持 SKILL.md 正文在 500 行以内，只写 Claude 缺少的领域知识、约束和步骤。
4. 根据任务出错代价和路径是否唯一，选择高、中、低自由度：精确脚本、参数化脚本或文本原则。
5. 将详细参考、表单指南、示例和脚本拆分为独立文件，形成渐进式披露结构，并避免深层嵌套引用。
6. 在计划使用的所有模型上测试，针对 Haiku 补充更多引导，对 Opus 避免过度解释。

## 关键点

- description 是技能发现的关键：Claude 启动时只预加载所有技能的 name 和 description，再根据用户请求选择读取 SKILL.md；描述模糊会直接导致选错或不选。
- 简洁的核心不是删字数，而是默认模型已具备通用知识，只保留与当前任务有关的外部知识、约束和特殊流程。
- 自由度要匹配任务脆弱性：数据库迁移、合规操作等“窄桥”场景应给低自由度的精确步骤；代码评审、开放分析等“开阔地”场景应给高自由度原则。
- 渐进式披露能显著降低上下文成本：SKILL.md 作为目录，详细内容放入参考文件按需加载，而不是把全部细节塞进主文件。
- 技能效果依赖底层模型，同一个 Skill 在 Haiku、Sonnet、Opus 上可能表现不同，必须用目标模型验证并调整说明细度。
- 避免深层嵌套引用：Claude 对引用文件可能只做部分读取或预览，嵌套引用容易造成信息丢失，应尽量保持引用扁平化。

## 对比与权衡

- 相比把所有规则直接写入系统提示或项目 prompt，Agent Skills 通过元数据预加载和按需读取文件，能在大规模技能库中保持低常驻 token 和低干扰，但要求 description 写得足够精确，否则发现阶段就会失败。
- 相比 MCP 工具连接外部 API/数据源，Skills 更偏静态领域知识和标准流程的“教 Claude 怎么做”，开发轻量、易版本化管理；但 Skills 不直接提供实时数据或外部执行能力，需要时仍要与 MCP 配合。
- 相比直接把脚本或规则作为一次性 prompt 粘贴给模型，Skill 将可复用的操作流程和知识固化为可发现能力包，更适合团队共享和长期维护，但需要额外设计目录结构和命名规范。

## 自测问题

**问: 为什么 description 对 Skill 发现最重要？**

Claude 在启动时把所有技能元数据中的 name 和 description 预加载到上下文，但 SKILL.md 正文只有在选中后才读取。因此 description 必须说明 what 和 when，覆盖用户请求中可能出现的关键词；如果只写“帮助处理文件”这类模糊内容，在 100+ 技能的竞争选择中很容易漏选或误选。

**问: 如何设计渐进式披露的目录结构？**

SKILL.md 保持 500 行以内，仅作为总览和导航；把表单填写、API 参考、示例等拆成 FORMS.md、reference.md、examples.md，脚本放 scripts/ 目录只执行不加载。多个业务域可以按域建 reference/finance.md、sales.md，这样模型只读取与当前问题相关的文件。还要避免文件之间层级过深的引用，因为模型可能用 head 等命令只读前几行。

**问: 任务自由度的高、中、低如何选择？**

关键看任务容错率和路径唯一性。高风险且需要严格一致的操作，如数据库迁移，必须给低自由度的具体脚本和顺序；存在首选模式但允许参数变化的场景，给中自由度的带参脚本或伪代码；多路径都合理、依赖上下文的场景，只给文本原则。类比窄桥和开阔地：越窄越危险，指令要越精确。

**问: 同一个 Skill 为什么要在多个模型上测试？**

Skill 本质是对模型能力的补充，模型基础能力不同。Haiku 速度快成本低但需要更多明确引导；Sonnet 平衡；Opus 推理强，过度解释会浪费上下文甚至干扰判断。所以一个 Skill 在 Opus 上完美，在 Haiku 上可能缺少足够指引；跨模型测试后应调整到各组都可用，或在文档中标注模型差异。

**问: 写 SKILL.md 时如何真正做到“简洁”？**

采用 token 审计视角：每段都问“Claude 是否已经知道这个？”“这段能否删掉或移到按需文件？”默认 Claude 知道 PDF、库、基础代码等通用知识，只补充本项目特有的 schema、命名、流程、约束、坑。SKILL.md 是目录不是教程；超过 500 行就拆分文件。

## 适用场景

- 企业搭建多个内部技能库（如 PDF 处理、BigQuery 查询、Git 提交规范）时，统一 name/description 和目录结构，提升发现率。
- 需要把标准作业流程或高风险操作（如数据库迁移、合规表单填写）固化为可复用 Skill，并让模型按精确步骤执行。
- 在多个模型档位（Haiku/Sonnet/Opus）上部署同一个 Agent 能力，需要测试不同引导细度并控制上下文成本。
- 已有复杂领域资料（API 参考、表单规则、示例）需要包装为 Agent 能力，而不是每次对话都临时粘贴大段 prompt。

## 标签

`Claude` `Agent Skills` `提示工程` `上下文工程` `AI Agent`

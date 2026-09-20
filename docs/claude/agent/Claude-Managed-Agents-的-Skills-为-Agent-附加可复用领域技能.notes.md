# Claude Managed Agents 的 Skills：为 Agent 附加可复用领域技能

*原文: [https://platform.claude.com/docs/en/managed-agents/skills](https://platform.claude.com/docs/en/managed-agents/skills) · 来源: web · 生成时间: 2026-09-20T07:19:21.565060+00:00*

## 背景

通用大模型 Agent 在垂直任务中常缺乏稳定的领域工作流和工具约定，工程师过去只能在每次会话中反复注入长指令或脚本。Claude Managed Agents 需要一种可版本化、可共享、按需加载的方式把通用代理转成文档处理、代码审查等专业代理。Skills 由此被设计为基于文件系统的领域知识包，用 SKILL.md 作为入口，把领域说明和支撑文件统一管理。

## 痛点

如果没有 Skills，团队需要在每个 agent 或会话中手动复制领域指令和脚本，既浪费上下文又难以保证一致性。多代理和多仓库之间无法自动共享最佳实践，agent 也无法根据任务动态发现并调用可复用的工作流。

## 解决办法

Skills 本质是给 agent 的可插拔专家手册。每个 skill 是一个包含 SKILL.md 的目录，SKILL.md 中的 name 和 description 用于让模型判断何时调用，真正的指令和资源在任务相关时由 agent 读取。附加方式有两种：一是在创建 agent 时通过 skills 数组指定 type、skill_id 和 version；二是挂载 GitHub 仓库，session 启动时扫描 .claude/skills/<skill-name>/SKILL.md 自动发现。版本号可 pin 或跟随 latest；ant apply 会写 claude-lock.json 以保证后续更新是同一 skill 的新版本而非重复创建。每个 skill 只注入少量元数据和指令到上下文，详细内容按需读取。

## 关键代码示例

```json
{
  "agent": {
    "name": "doc-reviewer",
    "skills": [
      { "type": "anthropic", "skill_id": "xlsx", "version": "latest" },
      { "type": "custom", "skill_id": "skill_01ABC123", "version": "3" }
    ],
    "tools": [{ "type": "read" }]
  },
  "github_repository": {
    "repo": "acme/repo",
    "checkout": "main",
    "mount_path": "/workspace/acme-repo"
  }
}
```

这段示意配置展示了 Skills 的两种来源。skills 数组中，anthropic 类型用短名 xlsx 直接引用预构建技能，custom 类型用创建后返回的 skill_* ID 并 pin 到版本 3；tools 中启用 read 是 GitHub 仓库技能发现的依赖。github_repository 挂载让 session 启动时扫描仓库根 .claude/skills 目录，无需把仓库技能写进 skills 数组。

## 关键流程

1. 自定义技能：创建包含 SKILL.md 和支持文件的目录，上传到 workspace 或使用 ant apply 推送并写入 claude-lock.json，获得 skill_id。
2. 附加技能：创建 agent 时在 skills 数组中声明 type、skill_id 和 version；预构建技能用短名，自定义技能用 skill_* ID。
3. 仓库加载：确保仓库根存在 .claude/skills/<skill-name>/SKILL.md 且仅一层目录，创建 session 挂载 github_repository 并启用 read 工具。
4. 版本与更新：自定义技能通过 claude-lock.json 复用 skill_id 更新版本；仓库技能跟随 session 启动时的 checkout 状态，更新后需新 session 生效。

## 关键点

- Skills 把领域知识从 prompt 中解耦为可复用、可版本化的文件系统目录；这让最佳实践能够像代码一样被管理、审查和分享。
- 每个 skill 会以元数据和指令的形式占用少量上下文窗口，因此 session 有 500 个去重技能上限；架构设计时需要在专业化深度和上下文成本之间平衡。
- 预构建 Anthropic skills 覆盖 docx、xlsx、pptx、pdf 等常见文档任务，无需创建和上传，直接通过短名引用；适合快速获得文档处理能力。
- 自定义技能由 SKILL.md 的 name 和 description 驱动自动触发，agent 仅在相关任务中读取完整内容；这就是它能降低不必要上下文消耗的核心机制。
- GitHub 自动发现严格依赖 .claude/skills/<skill-name>/SKILL.md 这个一层目录结构和默认 read 工具；目录层级错误或禁用 read 都会导致技能不加载。
- 仓库技能的扫描只发生在 session 启动时，并基于 checkout 分支或 commit；团队需要把更新技能需新建会话纳入发布流程。

## 对比与权衡

- 相比在 system prompt 中写死长段指令，Skills 通过 description 触发按需读取，能显著减少每个 session 的不相关上下文，但它的自动调用依赖模型判断，复杂任务可能需要额外提示。
- 相比 MCP 这类外部工具协议，Skills 更偏静态专家知识和脚本，不要求运行时外部服务，适合随仓库版本化；但它在实时系统交互和强结构化工具调用上不如 MCP。
- 相比直接上传单文件脚本或知识文件，Skills 以 SKILL.md 为入口并允许多个支持文件组成目录，可携带 scripts 和 resources；这让领域知识更有结构，但也增加了目录规范要求。

## 自测问题

**问: Skills 和 system prompt 里的长指令有什么本质区别？**

从上下文成本、可复用性、版本化和触发方式回答。system prompt 每次全量注入，所有会话都消耗；Skills 只在 session 注入 name、description 和 metadata，完整指令在任务相关时读取。Skills 还能随仓库版本化、共享审计。

**问: 自定义 skill 的 skill_id 是怎么来的？如何做版本更新？**

创建包含 SKILL.md 和支持文件的目录并上传 workspace 后，API 返回 skill_* ID；使用 ant apply 会记录 claude-lock.json，下次 apply 根据 lock 更新为同一 ID 的新版本。附加时 version 可 pin 具体版本或 latest。

**问: 如果一个仓库里有 .claude/skills/tools/code-review/SKILL.md，为什么不会被自动发现？**

发现规则要求正好是 .claude/skills/<skill-name>/SKILL.md 的一层目录；多一层就不在 session 启动扫描的声明范围。可以改成 .claude/skills/code-review/SKILL.md，或让 agent 在读取子树时手动发现。

**问: 多个 agent 在同一个 session 中共享 skills 时，数量限制如何计算？**

上限 500 是按 session 内所有 agent 的 skills 去重后计算，不是每个 agent 单独 500；因此多 agent 编排时共享同一技能不会重复计数，但不同技能总数不能超过 500。

**问: 为什么说每个 skill 都有上下文成本？如何优化？**

skills 数组中的每个条目都会给模型增加元数据和简短指令，帮助模型判断何时使用；完整 SKILL.md 只在需要时读取。优化方法是控制 skill 数量、给 description 写清触发边界、把大技能拆成小技能、避免加载无关 skill。

## 适用场景

- 需要 agent 稳定处理 Excel、PowerPoint、PDF 等标准办公文档时，直接附加预构建 Anthropic skills。
- 把团队内部的代码审查、发布检查、测试流程沉淀为仓库 .claude/skills，让每个挂载该仓库的会话自动获得相同规范。
- 多 agent 协作场景中，为不同 agent 挂不同技能组合，例如文档处理 agent、代码安全 agent、数据分析 agent。
- 需要审计或版本化领域知识时，将 skill 目录纳入 Git 管理，通过分支或 commit 控制不同版本。

## 标签

`Claude` `Agent Skills` `Managed Agents` `SKILL.md` `Agent Engineering`

# Claude 定时部署（Scheduled Deployments）：用 Cron 自动运行 Agent

*原文: [https://platform.claude.com/docs/en/managed-agents/scheduled-deployments](https://platform.claude.com/docs/en/managed-agents/scheduled-deployments) · 来源: web · 生成时间: 2026-09-20T07:48:36.135726+00:00*

## 背景

当企业需要 Claude Agent 在固定时间自主完成任务（如每日报告、定期巡检）时，手动在控制台逐个启动会话既不现实也不可靠。Scheduled deployments 把 agent 配置、环境、会话初始事件和调度策略统一封装，由 Claude API 在后台自动创建会话。它的本质是给托管 Agent 加了一个可追踪的定时触发层，类似无人值守的 cron job，但具备成熟的错误记录、预算和 webhook 语义。

## 痛点

没有这个能力时，团队通常要自己写 cron 脚本调用 SDK，自己维护失败重试、超时、预算封顶和状态追踪，容易遗漏或造成成本失控。如果直接用手动触发，则无法保证固定节奏，且很难审计每次运行是否成功。

## 解决办法

创建 deployment 时，把调度定义为标准 POSIX cron 表达式和 IANA 时区，并指定每次会话所需的 agent、environment、初始事件（如 user.message），可选挂接 files/GitHub/memory/vault。每次触发都会生成一条 deployment run 记录：成功时带 session_id，失败时带结构化 error.type；同时实际执行会加入少量 jitter 分散平台负载。预算不是部署级累计，而是把上限复制到每个新 session，类似于每次运行发一张固定面额消费卡。生命周期通过 pause/unpause/archive 控制，所有变化都有 webhook 事件。

## 关键代码示例

```python
import requests

# 创建定时部署：工作日早上 9 点美东时间触发
resp = requests.post(
    "https://api.anthropic.com/v1/deployments",
    headers={
        "Authorization": "Bearer $ANTHROPIC_API_KEY",
        "Content-Type": "application/json",
        "anthropic-version": "2023-06-01",
    },
    json={
        "schedule": {
            "expression": "0 9 * * 1-5",     # 标准 POSIX cron：分 时 日 月 周
            "timezone": "America/New_York", # IANA 时区，DST 按当地墙上时间
        },
        "agent_id": "agent_...",
        "environment": {"type": "cloud", "id": "env_..."},
        "sessions": [
            {"type": "user.message",
             "content": "Summarize overnight support tickets and draft a report."}
        ],
        "budget": {"type": "list_cost", "limit": 2000},  # 每次运行约 $20 上限
    },
)
print(resp.json()["schedule"]["upcoming_runs_at"])
```

这段代码创建了一个 scheduled deployment：schedule 表示工作日上午 9 点触发，时区为美东；sessions 提供初始 user.message，告诉 agent 本次要做什么；budget 限制每次运行成本。返回的 upcoming_runs_at 用于校验调度是否按预期生效，这是上线前必须检查的字段。

## 关键流程

1. 准备 agent 配置与目标环境：确定是 cloud 还是 self-hosted，并按需准备 files/GitHub/memory/vault。
2. 创建部署：调用 Deployments API 传入 schedule（cron 表达式 + IANA 时区）、session 配置、至少一个初始事件以及可选 budget。
3. 校验调度：从创建响应的 upcoming_runs_at 检查接下来的触发时间；记住实际执行可能有最多 9 分钟抖动。
4. 监控运行：通过 List Deployment Runs 或 webhook 跟踪每个 deployment run，成功时拿 session_id，失败时看 error.type。
5. 运维生命周期：需要暂停时 pause，恢复时 unpause；确认不再使用后 archive；注意某些永久错误会自动 pause 或 archive。

## 关键点

- 标准 POSIX cron 的语义是字面墙上时间匹配，配合 IANA 时区，DST 切换不会改变触发时刻；这是排查“为什么没有在 UTC 时间变化”的关键。
- 每次调度触发都会生成一条 deployment run 记录，成功时包含 session_id，失败时包含 error.type；先查 run 再查 session，是定位定时任务问题的高效顺序。
- deployment 的 budget 是按每次运行复制到新 session，不是所有运行累计共享；因此每次运行都可能花到上限，需要结合使用频率估算总成本。
- 调度实际执行带 jitter，避免所有租户在同一 cron 边界同时创建会话；监控和 SLA 要容忍最多 9 分钟或 15% 间隔的随机延迟。
- pause 只停止未来触发，已运行 session 继续；archive 是终态，不可恢复；自动 pause/archive 机制可以防止坏配置反复失败。
- 生命周期和运行结果都有 webhook 事件，应该用事件驱动告警代替轮询，尤其适合大量部署的运维场景。

## 对比与权衡

- 相比自建 cron 脚本调用 Claude SDK，Scheduled deployments 在运行记录、错误分类、预算、webhook 和生命周期管理上更完善，减少自己写胶水代码；但自定义重试、依赖前序任务或复杂分支等灵活性不如自建调度框架。
- 相比 Airflow/Temporal 等工作流调度器，它更轻量、无需维护 worker，适合单 agent 固定节奏任务；但在 DAG 依赖、回填、动态触发和复杂重试策略上不如后者。
- 相比在控制台手动定期触发，它消除了人为遗忘并保证固定节奏，且每次运行可审计；但调度参数和资源绑定在平台内，跨多云或本地复杂集成场景可能受限。

## 自测问题

**问: deployment 的 budget 是累计上限还是每次运行独立？如果某次运行已经超限会怎样？**

每次运行独立。创建或更新 deployment 时，budget 会复制到每个新 session；session 自己达到 list_cost 上限后进入 budget_reached 暂停状态。修改 deployment budget 只影响之后新启动的 run，不会追溯修改已运行 session；运行中的 session 可以通过 session API 单独调整。

**问: cron 调度在 DST 切换时会提前或延后吗？为什么？**

不会，因为使用字面墙上时间匹配。比如 America/New_York 的 `0 20 * * *` 在 EST 和 EDT 都是晚上 8 点本地时间。这种语义适合本地时间敏感的任务，但如果外部系统按 UTC 记录，需要注意时区偏移变化。

**问: 实际触发时间为什么会有 jitter？最大是多少？**

平台为了分散负载，避免所有定时任务在同一秒同时创建 session，会加入随机抖动。文档给出最大为调度间隔的 15%，最小 5 秒，最大 9 分钟。因此如果看到触发比 cron 时间晚几分钟，不一定是故障。

**问: pause、unpause 和 archive 有什么区别？哪些错误会触发自动 pause 或 archive？**

pause 停止未来调度但保留配置，已运行 session 继续，manual run 也可用；unpause 从下次调度恢复且不补跑遗漏；archive 是终态。Agent 归档/删除通常导致 deployment 自动 archive；子 agent 被归档、environment 或 vault 不可恢复等会记录 failed run 并自动 pause，便于修复后恢复。

**问: 一次定时任务没有触发或失败了，如何系统性排查？**

先看 deployment 是否被 pause/archive；再用 List Deployment Runs 过滤 error 查 failed run 的 error.type（如 session_rate_limited_error、environment_archived_error）；成功 run 会带 session_id，再沿 session event stream 或 webhook 查看 session 生命周期。还要注意 jitter 可能造成短时间延迟。

## 适用场景

- 每日或每周定时生成经营报告、客户反馈摘要或竞品监控简报。
- 在夜间对代码仓库做自动巡检、依赖升级安全检查或 issue 自动分类。
- 按固定节奏从外部数据源抓取数据、清洗后写入 Claude memory store 供后续会话使用。
- 在非工作时间运行需要隔离环境且可能花钱较多的 agent 任务，用每次运行 budget 控制单次成本。

## 标签

`Scheduled Deployments` `Claude API` `Cron` `Managed Agents` `自动化调度`

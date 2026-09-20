# 将 Agent 接入 GitHub：仓库挂载与 PR 工作流

*原文: [https://platform.claude.com/docs/en/managed-agents/github](https://platform.claude.com/docs/en/managed-agents/github) · 来源: web · 生成时间: 2026-09-20T07:42:51.330247+00:00*

## 背景

在托管 Agent 或云沙箱环境中，会话通常是临时的，Agent 需要访问 GitHub 代码才能完成代码理解、修改和提交等任务。过去直接在 Agent 里写 git clone 或调用 GitHub API 会带来凭证管理混乱、路径不统一、每次启动慢等问题。声明式资源挂载和 GitHub MCP server 正是为了解决这些场景而出现。

## 痛点

如果没有这个机制，开发者需要手动将仓库克隆进沙箱，或者把 GitHub token 硬编码在 Agent 逻辑里，容易泄露且难以轮换。临时会话每次重新克隆仓库耗时较长，而且多个仓库或分支的上下文管理会很混乱。

## 解决办法

核心做法是把 GitHub 仓库作为会话资源声明，平台在会话启动时自动克隆到 /workspace 下的指定路径，并对仓库做缓存以加速后续会话。认证 token 只在该资源声明中传入，平台不回显，从而把凭证作用域限制在单个仓库上。同时 agent 通过 GitHub MCP server 获得创建分支、提交、推送等 API 工具，形成“文件系统直接改代码 + MCP 操作远程仓库”的组合。checkout 字段支持指定分支或 commit，保证会话基于可复现的代码状态。多仓库只需在 resources 数组增加条目，运行中只能轮换 token 而不能增删仓库，以保持会话隔离性。

## 关键代码示例

```json
{
  "resources": [
    {
      "type": "github_repository",
      "url": "https://github.com/acme/my-repo",
      "authorization_token": "github_pat_xxx",
      "mount_path": "/workspace/my-repo",
      "checkout": { "type": "branch", "name": "main" }
    },
    {
      "type": "github_repository",
      "url": "https://github.com/acme/another-repo",
      "authorization_token": "github_pat_yyy",
      "mount_path": "/workspace/another-repo",
      "checkout": { "type": "commit", "sha": "abc123" }
    }
  ]
}
```

这段 JSON 是创建会话时挂载 GitHub 仓库的核心配置。resources 数组中每一项声明一个仓库：url 必须是 HTTPS 且不带 .git，authorization_token 用于克隆且不会被回显，mount_path 决定仓库在沙箱中的位置，checkout 指定分支或 commit 以固定代码状态。多仓库可以像第二个条目一样继续追加，并使用独立 token。

## 关键流程

1. 在 agent 定义中声明 GitHub MCP server 的 URL，但不包含认证 token。
2. 创建 session 时在 resources 数组添加 github_repository 资源，填写 url、authorization_token、可选 mount_path 和 checkout。
3. 根据操作范围使用最小权限 token：克隆私有仓库和创建 PR 用 repo，公开仓库读 issue 用 public_repo。
4. 如需多个仓库，在 resources 数组添加多个资源条目，每个可以配置独立 token 和挂载路径。
5. 运行中通过 resources.list 查看资源 id，并可以 rotate token；若要更换挂载仓库，需要新建 session。
6. Agent 通过 GitHub MCP 工具创建分支、提交、推送，并打开 Pull Request。

## 关键点

- 仓库资源必须使用 HTTPS URL 且不带 .git 后缀，这降低了歧义，便于平台统一解析、校验和安全策略。
- authorization_token 只在挂载资源时用于克隆，平台不会在 API 响应中回显，降低泄露风险。
- 仓库会被缓存，后续相同仓库的会话启动更快；但会话基于启动时 checkout 的快照，远端后续更新不会自动同步。
- token 应遵循最小权限：私有仓库克隆和创建 PR 用 repo，公开仓库读 issue 用 public_repo，避免使用过大的 scope。
- 多仓库通过 resources 数组声明，每个仓库可拥有独立 token 和挂载路径；运行中可轮换 token，但更换挂载仓库需要新建会话。
- 仓库根目录 .claude/skills 中的技能会在会话启动时自动加载，适合团队共享 Agent 能力和工作流。

## 对比与权衡

- 相比在 agent 代码里直接执行 git clone，资源挂载方式统一管理凭证、路径和缓存，避免 agent 接触明文凭证，但灵活性更低，不能运行时随意克隆任意仓库。
- 相比单独使用普通 GitHub MCP server 只提供 API 工具，挂载仓库到沙箱文件系统让 agent 能直接读/改文件再结合 API 提交，更适合“读代码-改代码-提 PR”闭环；但需要额外的存储和缓存管理。
- 相比 GitHub Actions/CI，这种会话式挂载提供交互式、多步推理的 agent 工作流，而不是事件驱动的固定流水线；但资源生命周期与 session 绑定，不适合长期常驻任务。

## 自测问题

**问: 为什么同一个 GitHub 集成需要同时声明 MCP server 和挂载 repository resource？**

MCP server 提供远程 Git/GitHub 操作工具，如 create_branch、commit、push、create_pull_request，但不负责把代码放进沙箱文件系统；resource 负责让本地文件可读可写。两者组合才能让 agent 直接修改 /workspace 下的文件后，再用 MCP 工具推到远端。没有 resource 时 agent 只能通过 API 看代码，不方便全局搜索、运行测试或批量修改。

**问: 仓库缓存是怎么实现的？会有什么坑？**

平台会按 URL 缓存克隆后的 objects 或工作区，后续新会话挂载相同仓库时快速恢复，而不是从零 clone。坑在于缓存可能是启动时的快照，如果远端分支已经推进，旧会话不会自动更新；需要新会话或明确 checkout 到新 commit 来获取最新代码。此外缓存失效策略和磁盘占用也是需要考虑的问题。

**问: token 为什么要放在 resource 而不是 agent 定义的 MCP server 配置里？**

MCP server 是全局工具服务，可能被多个仓库或会话共享；如果把 token 放在 server 配置中，凭证作用域过大，且 agent 定义通常会展示或存储。放在 resource 上让 token 与具体仓库绑定，使用最小权限，降低泄露风险，也便于单独 rotate 某个仓库的 token。

**问: 如果需要同一会话操作多个仓库，且它们需要不同的 token，怎么做？**

在 resources 数组中为每个仓库添加独立条目，各自指定 url、authorization_token 和 mount_path。不同 token 可以对应不同的 GitHub 账号或权限范围。注意 mount_path 必须唯一，否则会冲突；agent 通过不同路径区分仓库。

**问: 为什么强制 HTTPS URL，不接受 SSH？**

HTTPS 更容易配合 token 进行认证，适合在 ephemeral sandbox 中安全注入凭证；SSH 需要在沙箱里管理私钥或部署密钥，增加复杂度和泄露风险。统一 URL 形式也方便平台校验、缓存和生成资源标识。

## 适用场景

- 自动化代码修复或重构：agent 读取仓库代码，修改后自动创建 Pull Request。
- 跨仓库变更：同时挂载多个微服务或前后端仓库，让 agent 做全链路修改。
- 可复现的代码审查环境：通过 checkout 指定 commit 或分支，确保审查基于特定代码快照。
- 团队技能共享：仓库根目录 .claude/skills 自动加载，让 Agent 具备团队自定义能力。

## 标签

`GitHub` `MCP` `Agent 沙箱` `仓库挂载` `Pull Request`

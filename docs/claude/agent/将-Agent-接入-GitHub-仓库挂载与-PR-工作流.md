---
title: 将 Agent 接入 GitHub：仓库挂载与 PR 工作流
url: https://platform.claude.com/docs/en/managed-agents/github
source_type: web
folder: claude/agent
author: null
tags:
- GitHub
- MCP
- Agent 沙箱
- 仓库挂载
- Pull Request
summary: 在云沙箱中通过声明式资源挂载 GitHub 仓库，并结合 MCP 工具实现代码读取、分支提交与 Pull Request 创建。
fetched_at: '2026-09-20T07:42:51.330247+00:00'
---

Connect your agent to GitHub repositories for cloning, reading, and creating pull requests.

You can mount a GitHub repository to your session sandbox and connect to the GitHub MCP for making pull requests.

GitHub repositories are cached, so future sessions that use the same repository start faster.

## GitHub MCP and session resources

First, create an agent that declares the GitHub MCP server. The agent definition holds the server URL but no authentication token:

Then create a session that mounts the GitHub repository:

A `github_repository` resource accepts the following fields:

| Field | Description |
| --- | --- |
| `type` | Required. Must be `"github_repository"`. |
| `url` | Required. The repository's HTTPS URL in the form `https://github.com/<owner>/<repo>`, without a `.git` suffix. Other forms, including SSH URLs, are rejected with an `invalid_request_error`. |
| `authorization_token` | Required. The GitHub token used to clone the repository. It is not echoed in API responses. See [Token permissions](https://platform.claude.com/docs/en/managed-agents/github#token-permissions). |
| `mount_path` | Optional. The directory under `/workspace` to clone the repository into. Defaults to `/workspace/<repo-name>`. |
| `checkout` | Optional. A branch (`{"type": "branch", "name": "main"}`) or commit (`{"type": "commit", "sha": "..."}`) to check out. Defaults to the repository's default branch. |

Mounting a repository also loads any skills stored in its root `.claude/skills` directory. Skills are discovered once per session, from the repository state checked out at session start. See [Load skills from a GitHub repository](https://platform.claude.com/docs/en/managed-agents/skills#load-skills-from-a-github-repository).

## Token permissions

When providing a GitHub token, use the minimum required permissions:

| Action | Required scopes |
| --- | --- |
| Clone private repos | `repo` |
| Create PRs | `repo` |
| Read issues | `repo` (private) or `public_repo` |
| Create issues | `repo` (private) or `public_repo` |

## Multiple repositories

Mount multiple repositories by adding entries to the `resources` array:

## Managing repositories on a running session

After a session is created, you can list its repository resources and rotate their authorization tokens. Each resource has an `id` returned at session creation time (or through `resources.list`) that you use for updates. Repositories are attached for the lifetime of the session; to change which repositories are mounted, create a new session.

## Creating pull requests

With the GitHub MCP server, the agent can create branches, commit changes, and push them:

## Next steps

Stream events and steer the agent while it opens the pull request

Connect more MCP servers to give the agent additional tools

Mount files in the sandbox alongside your repositories
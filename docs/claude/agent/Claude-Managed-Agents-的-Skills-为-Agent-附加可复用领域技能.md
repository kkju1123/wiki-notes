---
title: Claude Managed Agents 的 Skills：为 Agent 附加可复用领域技能
url: https://platform.claude.com/docs/en/managed-agents/skills
source_type: web
folder: claude/agent
author: null
tags:
- Claude
- Agent Skills
- Managed Agents
- SKILL.md
- Agent Engineering
summary: Skills 将领域工作流、上下文和最佳实践打包为基于文件系统的可复用资源，可在 Claude Managed Agents 中手动附加或从 GitHub
  自动发现。
fetched_at: '2026-09-20T07:19:21.565060+00:00'
---

Attach pre-built or custom skills to an agent in Claude Managed Agents to give it reusable, filesystem-based expertise for domain-specific workflows.

Skills are reusable, filesystem-based resources that give your agent domain-specific expertise: workflows, context, and best practices that turn a general-purpose agent into a specialist. Each skill you add incurs a modest cost on the session's context window, adding instructions and metadata that help the model use the skill. Learn more in the [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) overview.

Skills reach your agent in two ways: attach them through the agent's `skills` array, or [load them from a GitHub repository](https://platform.claude.com/docs/en/managed-agents/skills#load-skills-from-a-github-repository) mounted on the session. Attached skills come in two types. All skills work the same way: your agent invokes them automatically when they are relevant to the task.

*   **Pre-built Anthropic skills:** Common document tasks such as PowerPoint, Excel, Word, and PDF handling (`pptx`, `xlsx`, `docx`, `pdf`).
*   **Custom skills:** Skills you author and upload to your workspace.

To learn how to author custom skills, see [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) and [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices). To upload a custom skill to your workspace, see [Create a custom skill](https://platform.claude.com/docs/en/managed-agents/skills#create-a-custom-skill).

## Create a custom skill

A custom skill is a directory containing a `SKILL.md` file plus any supporting files, uploaded to your workspace as a zip archive or as individual files. Creating the skill returns the `skill_*` ID you reference when attaching it to an agent. Anthropic pre-built skills are already available in every workspace and don't require this step. To use only pre-built skills, skip to [Attach skills to an agent](https://platform.claude.com/docs/en/managed-agents/skills#attach-skills-to-an-agent).

These examples omit the optional `display_name` field, so the skill's display name is derived from the `name` field in `SKILL.md`. An explicit `display_name` can be up to 255 characters and doesn't need to be unique within your workspace.

[`ant apply`](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/apply) uploads the `skills/pr-summary` directory, prints the new skill's ID, and records it in `claude-lock.json`. Commit `claude-lock.json` so the next `ant apply` uploads your edits as a new version instead of creating a second skill.

To list, retrieve, delete, and version custom skills, see [Managing custom skills](https://platform.claude.com/docs/en/build-with-claude/skills-guide#managing-custom-skills). For the full request and response schemas, see the [Create Skill API reference](https://platform.claude.com/docs/en/api/skills/create). Skill bundles upload directly to the Skills API rather than through the [Files API](https://platform.claude.com/docs/en/build-with-claude/files).

## Attach skills to an agent

Attach skills when creating an agent. Each [session](https://platform.claude.com/docs/en/managed-agents/sessions) supports up to 500 skills, counted as the deduplicated set across every agent in the session (see [Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)).

Each entry in the `skills` array uses the following fields:

| Field | Description |
| --- | --- |
| `type` | Either `anthropic` for pre-built skills or `custom` for workspace-authored skills. |
| `skill_id` | The skill identifier. For Anthropic skills, use the short name (for example, `xlsx`). For custom skills, use the `skill_*` ID returned at creation (see [Create a custom skill](https://platform.claude.com/docs/en/managed-agents/skills#create-a-custom-skill)). |
| `version` | Pin to a specific version or use `latest`. Optional. Defaults to `latest` when omitted. Applies to both Anthropic and custom skills. |

## Load skills from a GitHub repository

Skills can also live in your codebase. When a session mounts a repository through the [`github_repository` resource](https://platform.claude.com/docs/en/managed-agents/github), the repository's root `.claude/skills` directory is scanned at session start, and each skill found there becomes available to the agent. No upload and no entry in the agent's `skills` array are required. The agent sees each discovered skill's name, description, and path in the sandbox, and reads the skill's `SKILL.md` when a task matches, including any scripts and resources the skill ships. Discovery relies on the agent's `read` tool from the [agent toolset](https://platform.claude.com/docs/en/managed-agents/tools), which is enabled by default; an agent with `read` disabled doesn't load repository skills.

Discovery finds skills at exactly `.claude/skills/<skill-name>/SKILL.md`, one directory level deep at the repository root:

*   `your-repo/`
    *   `.claude/`
        *   `skills/`
            *   `code-review/`
                *   `SKILL.md`

            *   `release-process/`
                *   `SKILL.md`
                *   `scripts/`
                    *   `run_checks.sh`

    *   `src/`

Locations that don't match this layout aren't discovered at session start:

*   `.claude/skills/SKILL.md`: a `SKILL.md` with no skill directory around it
*   `.claude/skills/tools/code-review/SKILL.md`: nested more than one directory level deep
*   `skills/code-review/SKILL.md`: a `skills` directory outside `.claude`

A `.claude/skills` directory elsewhere in the repository, such as inside a package subdirectory, isn't announced at session start; those skills can still surface when the agent reads files under that subtree.

Repository skills use the same `SKILL.md` format as the custom skills you upload. For the format and authoring guidance, see [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) and [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

To load skills from a repository, create a session that mounts it. This is the same request shown in [Accessing GitHub](https://platform.claude.com/docs/en/managed-agents/github#token-permissions); `mount_path` is optional and defaults to `/workspace/<repo-name>`:

For private repositories, the resource's `authorization_token` must have access to the repository. This is the same personal access token flow used for any repository mount; see [Accessing GitHub](https://platform.claude.com/docs/en/managed-agents/github#token-permissions).

Discovered skills follow the checked-out state of the repository: the `checkout` branch or commit when the resource sets one, otherwise the repository's default branch. The scan runs once, when the session starts. Commits pushed mid-session are not picked up; to load updated skills, start a new session.

Repository skills work alongside skills attached through the agent's `skills` array. If a repository skill shares a name with an attached skill, or with a skill from another mounted repository, both are available; each is announced with its own path.

## Next steps

Customize cloud sandboxes for your sessions.

Learn how to use Agent Skills to extend Claude's capabilities through the API.

Upload files once and reference them across API requests.

Learn how to use Agent Skills to create documents with the Claude API in under 10 minutes.
---
title: Claude Bash Tool：持久 Shell 会话与客户端工具调用机制
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Bash Tool
- tool use
- 持久会话
- 客户端工具安全
summary: Bash tool 让 Claude 返回 shell 命令，由应用在持久 bash 会话中执行并回传输出，实现多步骤自动化任务。
fetched_at: '2026-09-20T03:57:49.876758+00:00'
---

Let Claude request shell commands that your application runs in a persistent bash session and returns as tool results.

The bash tool is a [client tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works): Claude doesn't run commands itself. When you include the tool in a request, Claude replies with a `tool_use` block that names the command to run. Your application runs that command in a bash session it owns and returns the output in a `tool_result` block.

Your application keeps one bash process alive across tool calls, so state persists between commands. The working directory, environment variables, and any files a command creates are still there for the next command.

The current version of the tool is `bash_20250124`. For model support, beta headers, and the earlier version, see [Tool versions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool#tool-versions). For all Anthropic-provided tools, see the [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference).

## Use cases

*   **Development workflows:** Run build commands, tests, and development tools
*   **System automation:** Execute scripts, manage files, automate tasks
*   **Data processing:** Process files, run analysis scripts, manage datasets
*   **Environment setup:** Install packages, configure environments

## Quick start

Claude responds with `stop_reason: "tool_use"` and a `tool_use` block that contains the command for your application to run:

Run `input.command` in your bash session and send the output back as a `tool_result`. See [Implement the bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool#implement-the-bash-tool) for the round trip.

## How it works

Each tool call is one round trip between Claude and your application:

1.   Claude returns a `tool_use` block containing the `command` to run.
2.   Your application runs the command in its bash session.
3.   Your application returns the command's output, stdout and stderr together, to Claude in a `tool_result` block.
4.   Claude either requests another command in the same session or responds with text.

Claude can also return several `tool_use` blocks in one response. Run them in order in the same session and return all of the results in one `user` message. See [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use).

The API is stateless. Nothing about your shell session travels between requests, so your application decides when the session starts, how long it lives, and when to restart it. For the full request and response cycle, see [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

## Parameters

A bash tool definition has two required fields, `type` and `name`, and the `name` must be `bash`. The tool is schema-less: you don't provide an `input_schema`, because the schema is built into Claude's model and can't be modified. The following table lists the input fields Claude sets when it calls the tool.

| Parameter | Required | Description |
| --- | --- | --- |
| `command` | Yes* | The bash command to run |
| `restart` | No | Set to `true` to restart the bash session |

*Required unless using `restart`

To handle `restart: true`, kill the shell process, start a new one, and return a `tool_result` that confirms the restart. A restarted session starts clean: the working directory, environment variables, and any running processes are gone.

## Tool versions

`bash_20250124` is the current version of the tool, and it requires no beta header. Every model from Claude Sonnet 3.7 ([retired](https://platform.claude.com/docs/en/about-claude/model-deprecations)) onward accepts it, including all current Claude models.

The original `bash_20241022` version works only with the October 2024 Claude Sonnet 3.5 model ([retired](https://platform.claude.com/docs/en/about-claude/model-deprecations)). Requests that use it need the `anthropic-beta: computer-use-2024-10-22` header, and the SDKs expose it only in their beta namespaces. New integrations should use `bash_20250124`.

## Example: Multistep automation

Claude can chain commands across tool calls to complete a multistep task:

The session maintains state between commands, so files created in step 2 are available in step 3.

## Implement the bash tool

Claude determines which command to run. Your application owns everything else: the shell process, the timeout, and the safety checks. The following steps show a minimal implementation.

1.   ### Create a persistent bash session

Start one long-lived bash process and run every command inside it. Because a pipe to a live process never reports end-of-file, the session prints a unique sentinel line after each command to mark where that command's output ends:

The session interleaves stderr with stdout, so error messages land where they happened. The example leaves out what a complete implementation also needs: a timeout that kills the shell and every process it started when a command hangs, then restarts the session. The [Use command timeouts](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool#follow-implementation-best-practices) best practice shows one way to add it. 
2.   ### Process Claude's tool calls

Extract and run commands from Claude's responses: 
3.   ### Return the result to Claude

Send the `tool_result` back in a `user` message that continues the same conversation. Claude either requests another command in the same session or finishes its answer:

Repeat the run-and-return cycle while `stop_reason` is `tool_use`. For the full loop, see [Handling results from client tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-results-from-client-tools). 
4.   ### Implement safety measures

Add validation and restrictions. Use an allowlist rather than a blocklist: a blocklist misses any command it didn't anticipate. The example also rejects shell operators that appear as separate words:

This check is a tripwire for obvious mistakes, not an enforcement boundary. It rejects the spaced chaining (`&&`), pipes, and redirection that the other examples on this page use. It does not catch an operator glued to a word, such as `cat data.txt|grep x`, because the tokenizer keeps `data.txt|grep` inside one token. Decide which commands and operators your application allows. The real control is isolation: run the whole session inside a container or a virtual machine (see [Security](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool#security)). 

### Handle errors

When a command fails or the session breaks, tell Claude what happened. Return the message as the `tool_result` content and set `is_error` to `true`, which marks the tool call as failed. See [Handling errors with is_error](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-errors-with-is-error).

### Follow implementation best practices

## Security

Beyond isolation, add these controls:

*   Validate commands before running them, with an allowlist rather than a blocklist. See [Implement the bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool#implement-the-bash-tool).
*   Set resource limits on the shell process (CPU, memory, and disk), for example with `ulimit`.
*   Log every command and its output so you can audit what ran.
*   Redact credentials and other secrets from output before returning it to Claude.

## Pricing

The bash tool definition adds the following input tokens to your request. This is in addition to the per-model [tool use system prompt](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#pricing) that applies whenever any tool is present.

| Model | Additional input tokens |
| --- | --- |
| Claude Opus 5, Claude Opus 4.8, and Claude Opus 4.7 | 325 tokens |
| Claude Opus 4.6, Claude Sonnet 4.6, and earlier | 244 tokens |

Additional tokens are consumed by:

*   Command outputs (stdout/stderr)
*   Error messages
*   Large file contents

See [tool use pricing](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#pricing) for complete pricing details.

## Common patterns

### Development workflows

*   Running tests: `pytest && coverage report`
*   Building projects: `npm install && npm run build`
*   Git operations: `git status && git add . && git commit -m "message"`

For guidance on using git as a checkpoint-and-recovery mechanism in long-running agent workflows, see [state management best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#state-management-best-practices).

### File operations

*   Processing data: `wc -l *.csv && ls -lh *.csv`
*   Searching files: `find . -name "*.py" | xargs grep "pattern"`
*   Creating backups: `tar -czf backup.tar.gz ./data`

### System tasks

*   Checking resources: `df -h && free -m`
*   Process management: `ps aux | grep python`
*   Environment setup: `export PATH=$PATH:/new/path && echo $PATH`

## Limitations

*   **No interactive commands:** The session can't run `vim`, `less`, password prompts, or any command that waits for input on stdin.
*   **No GUI applications:** The session is command-line only.
*   **Session scope:** Bash session state is client-side. Your application is responsible for maintaining the shell session between turns.
*   **Output limits:** The API doesn't truncate tool results (an oversized request is rejected). Truncate large outputs in your application before returning them to Claude.
*   **No streaming:** Output reaches Claude only when your application returns the `tool_result` in the next request.

## Combining with other tools

The bash tool pairs well with the [Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool): Claude edits a file with one tool and requests the command that runs it with the other.

## Next steps

View and modify text files to debug, fix, and improve code.

Connect Claude to external tools and APIs. See where tools execute, when Claude calls them, and which tool fits your task.
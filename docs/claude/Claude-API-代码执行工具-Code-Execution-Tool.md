---
title: Claude API 代码执行工具（Code Execution Tool）
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool
source_type: web
folder: claude
author: null
tags:
- Claude API
- Tool Use
- 代码执行
- 沙箱
- Agent
summary: Claude 在隔离沙箱中运行 Python/Bash 代码，完成数据分析、文件处理与精确计算，并通过容器复用保留状态，提升可靠性。
fetched_at: '2026-09-20T03:52:01.744326+00:00'
---

Run Python and bash code in a sandboxed container to analyze data, generate files, and iterate on solutions.

Claude can analyze data, create visualizations, perform complex calculations, run system commands, create and edit files, and process uploaded files directly within the API conversation. The code execution tool allows Claude to run Bash commands and manipulate files, including writing code, in a secure, sandboxed environment.

**Code execution is free when used with web search or web fetch (`web_search_20260209`, `web_fetch_20260209`, or later).** When one of those tools is in your request, there are no additional charges for code execution in that request beyond standard token costs. This covers both the code execution behind dynamic filtering and any code Claude runs directly. Standard code execution pricing applies when they are not included.

Code execution also powers dynamic filtering in the [web search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool) and [web fetch](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool) tools: Claude filters results inside the code execution environment before they reach the context window. When dynamic filtering runs, the API provisions the code execution it needs for the request automatically, so you don't add the code execution tool to your request for it.

## Tool versions

The code execution tool has three current versions, and every [supported model](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compatibility) accepts all three. Each version builds on the previous one:

*   `code_execution_20250825` supports Bash commands and file operations.
*   `code_execution_20260120` adds REPL state persistence and [programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) from within the sandbox. Claude Haiku 4.5 accepts the `code_execution_20260120` and `code_execution_20260521` tool types, but programmatic tool calling and the REPL state persistence that depends on it aren't available on it, so the newer versions behave like `code_execution_20250825` there.
*   `code_execution_20260521` is the same runtime as `code_execution_20260120`. The difference is that the tool description tells Claude about the 90-second wall-clock limit on each Python cell in programmatic tool calling, so Claude can budget long-running cells. A cell that exceeds the limit returns a normal code execution result with a non-zero `return_code` and a `detection_timeout` status message in its output. This is separate from the `execution_time_exceeded`[error code](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#errors), which the API returns when a whole tool invocation exceeds the maximum execution time.

None of the three tool versions requires an `anthropic-beta` header. The legacy code execution beta headers remain valid opt-ins.

The examples on this page use `code_execution_20250825`, which covers the Bash and file operations they demonstrate and behaves the same way on every supported model; use `code_execution_20260120` or later when you need programmatic tool calling or REPL state persistence. The current [web search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool) and [web fetch](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool) tools (`web_search_20260209`, `web_fetch_20260209`, and later) require `code_execution_20260120` or later as their code execution version.

Older tool versions aren't guaranteed to stay compatible with newer models. When you adopt a new model, check [Tool versions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#tool-versions) and [Compatibility](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compatibility), and prefer the newest tool version your integration supports.

## Quick start

Here's an example that asks Claude to perform a calculation:

The response interleaves `server_tool_use` blocks (the commands Claude ran) with their tool result blocks, followed by Claude's text. The top level also includes a `container` object whose `id` you can [reuse across requests](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#container-reuse). See [Response format](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#response-format) for the block shapes.

## How code execution works

When you add the code execution tool to your API request:

1.   Claude evaluates whether code execution would help answer your question
2.   The tool automatically provides Claude with the following capabilities:
    *   **Bash commands:** Run shell commands for system operations
    *   **File operations:** Create, view, and edit files directly, including writing code

3.   Claude can use any combination of these capabilities in a single request
4.   All operations run in a secure, sandboxed container. The container has no internet access, so Claude can't download packages at runtime: only the [pre-installed libraries](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#pre-installed-libraries) are available
5.   The API runs every command server-side and returns the results to Claude within the same request, so you never execute code or send back `tool_result` blocks yourself. One exception is when Claude calls one of your client tools alongside code execution: the API returns the code execution call without its result. The result arrives in a later response, after you send back the `tool_result` blocks for your client tools
6.   Each request runs in a new container unless you pass an earlier response's container ID back (see [Container reuse](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#container-reuse))
7.   Claude provides results with any generated charts, calculations, or analysis

The container has Python pre-installed. Claude writes Python with the file operations sub-tool and runs it with a Bash command. With `code_execution_20260120` or later and [programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling), the Python interpreter state (such as variable bindings) also persists across requests that reuse the container.

### When Claude runs code

Claude runs code when the request benefits from computation or file handling:

*   Non-trivial math (large numbers, many steps, precision-sensitive results)
*   Data analysis, file parsing, or visualization
*   Algorithm execution or simulation
*   Explicit requests to "run", "compute", or "execute"

Claude answers directly without running code for:

*   Simple arithmetic and well-known math facts
*   Factual, conversational, or creative requests
*   Simple unit conversions or translations

If you want Claude to run code for a borderline request, ask explicitly (for example, "run code to verify this").

## Work with files

### Upload and analyze your own files

To analyze your own data files (such as CSV, Excel, or images), upload them through the Files API and reference them in your request.

The Python environment can process various file types uploaded through the Files API, including:

*   CSV
*   Excel (.xlsx, .xls)
*   JSON
*   XML
*   Images (JPEG, PNG, GIF, WebP)
*   Text files (.txt, .md, .py, and others)

#### Upload and analyze files

1.   **Upload your file** using the [Files API](https://platform.claude.com/docs/en/build-with-claude/files)
2.   **Reference the file** in your message using a `container_upload` content block
3.   **Include the code execution tool** in your API request

### Retrieve generated files

When Claude saves files to its output directory during code execution (see [How generated files are captured](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#how-generated-files-are-captured)), each file's ID appears in the code execution tool result, and you can download it with the [Files API](https://platform.claude.com/docs/en/build-with-claude/files):

#### How generated files are captured

Each `bash_code_execution` call gets a new, empty directory, available to the command as `$OUTPUT_DIR`. When the command finishes, the files at the top level of that directory are captured and returned as the `file_id` entries in the result's `content` list. Files written anywhere else stay in the container and aren't returned.

The tool description tells Claude to share files by copying them into `$OUTPUT_DIR`. If your application depends on receiving a file, prompt Claude to copy it into `$OUTPUT_DIR` and list the directory in the same command, so the `ls` output confirms the capture (Claude doesn't see the `content` list):

A file Claude wrote elsewhere is still in the container, so you can [reuse the container](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#container-reuse) and ask Claude to copy it into `$OUTPUT_DIR`.

### Content Credentials on generated files

On the Claude API, supported image, video, and audio files that Claude produces in the code execution sandbox carry [C2PA](https://c2pa.org/) Content Credentials when you download them through the [Files API](https://platform.claude.com/docs/en/build-with-claude/files). [Supported formats](https://opensource.contentauthenticity.org/docs/sdk-repos/c2pa-python/docs/supported-formats/) include PNG, JPEG, GIF, WebP, TIFF, HEIC, AVIF, SVG, MP4, MOV, MP3, WAV, FLAC, and M4A. The credential is a cryptographically signed manifest embedded in the file's metadata. It identifies Anthropic as the issuer, carries a timestamp, and records the action description "Claude provided this file at the request of a user and may have created or modified the file contents."

Signing requires no changes to your requests or response handling, and the manifest records nothing about you, your organization, or your request. The file's visible content is unchanged. The manifest adds a few kilobytes, so the downloaded file's size and checksum differ from the file as it exists inside the container. Text files, PDFs, and office documents are not signed because they are not supported formats for signing. Files you upload are stored as-is, including any Content Credentials they already carry.

To verify a credential, inspect the file with any C2PA-compatible tool, such as the open-source [c2patool command-line utility](https://github.com/contentauth/c2pa-rs). Re-encoding, format conversion, screenshots, and tools that strip metadata remove the credential, so a missing credential doesn't mean a file wasn't produced with Claude. For more on why a credential can be missing, see [How Claude marks AI-generated content](https://support.claude.com/en/articles/16266773-how-claude-marks-ai-generated-content).

## Tool definition

The code execution tool requires no additional parameters:

Both fields are fixed: `type` selects the tool version, and `name` must be `code_execution`.

When you provide this tool, Claude automatically gains access to two sub-tools:

*   `bash_code_execution`: Run shell commands
*   `text_editor_code_execution`: View, create, and edit files, including writing code

When Claude runs code, the response also includes a top-level `container` object with the container's `id` and `expires_at` timestamp. Pass that ID back in the top-level `container` request parameter to keep using the same container. See [Container reuse](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#container-reuse).

## Response format

The code execution tool can return two types of results depending on the operation:

### Bash command response

### File operation responses

**View file:**

**Create file:**

**Edit file (str_replace):**

### Results

Bash command results (`bash_code_execution_result`) include:

*   `stdout`: Output from successful execution
*   `stderr`: Error messages if execution fails
*   `return_code`: 0 for success, non-zero for failure
*   `content`: A list with an entry for each file the command left in `$OUTPUT_DIR` (see [How generated files are captured](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#how-generated-files-are-captured)). Each entry carries the `file_id` to [retrieve the file](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#retrieve-generated-files) with the Files API

File operation results have their own fields:

*   **View** (`text_editor_code_execution_view_result`): `file_type`, `content`, `num_lines`, `start_line`, `total_lines`
*   **Create** (`text_editor_code_execution_create_result`): `is_file_update` (whether the file already existed)
*   **Edit** (`text_editor_code_execution_str_replace_result`): `old_start`, `old_lines`, `new_start`, `new_lines`, `lines` (diff format)

### Errors

Each tool type can return specific errors:

**Common errors (all tools):**

**Error codes by tool type:**

| Tool | Error code | Description |
| --- | --- | --- |
| All tools | `unavailable` | The tool is temporarily unavailable |
| All tools | `execution_time_exceeded` | The tool invocation exceeded the maximum execution time |
| All tools | `invalid_tool_input` | Invalid parameters provided to the tool |
| All tools | `too_many_requests` | Rate limit exceeded for tool usage |
| bash | `output_file_too_large` | Command output exceeded the maximum size |
| text_editor | `file_not_found` | File doesn't exist (for view/edit operations) |

An expired container can't be reused: requests that reference it return an error instead of restoring it. Send the request again without the `container` parameter to get a new container.

### `pause_turn` stop reason

The response might include a `pause_turn` stop reason, which indicates that the API paused a long-running turn. You may provide the response back as-is in a subsequent request to let Claude continue its turn, or modify the content if you want to interrupt the conversation.

## Containers

The code execution tool runs in a secure, containerized environment designed specifically for code execution, with a higher focus on Python.

### Runtime environment

*   **Python version:** 3.11
*   **Operating system:** Linux-based container
*   **Architecture:** x86_64 (AMD64)

### Resource limits

*   **Memory:** 5 GiB RAM
*   **Disk space:** 5 GiB workspace storage
*   **CPU:** 1 CPU
*   **Execution time:** A tool invocation that runs past the maximum execution time returns an `execution_time_exceeded`[error](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#errors). With [programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling), each REPL cell also has a 90-second wall-clock limit

### Networking and security

*   **Internet access:** Completely disabled for security
*   **External connections:** No outbound network requests permitted
*   **Sandbox isolation:** Full isolation from host system and other containers
*   **File access:** Limited to workspace directory only
*   **Workspace scoping:** Like the [Files API](https://platform.claude.com/docs/en/build-with-claude/files), containers are scoped to the request's workspace
*   **Expiration:** Containers expire 30 days after creation

### Pre-installed libraries

The sandboxed Python environment includes these commonly used libraries:

*   **Data science:** pandas, numpy, scipy, scikit-learn, statsmodels
*   **Visualization:** matplotlib, seaborn
*   **File processing:** pyarrow, openpyxl, xlsxwriter, xlrd, pillow, python-pptx, python-docx, pypdf, pdfplumber, pypdfium2, pdf2image, pdfkit, tabula-py, reportlab[pycairo], Img2pdf
*   **Math and computing:** sympy, mpmath
*   **Utilities:** tqdm, python-dateutil, pytz, joblib

The container also includes command-line tools such as unzip, unrar, 7zip, bc, rg (ripgrep), fd, and sqlite.

The container has no internet access, so Claude can't download or install additional packages at runtime: only the pre-installed libraries are available.

## Container reuse

You can reuse an existing container across multiple API requests by providing the container ID from a previous response. This allows you to maintain created files between requests. With `code_execution_20260120` or later and [programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling), the Python interpreter state persists as well.

Containers expire 30 days after creation. After about 5 minutes of inactivity a container is checkpointed, and sending a request with its ID inside the 30-day window restores it. The `expires_at` timestamp in the response's `container` object is a shorter rolling value and doesn't report the 30-day limit. A container that has expired can't be reused. Send the request again without the `container` parameter to get a new container.

### Example

## Using code execution with other execution tools

When you provide code execution alongside client-provided tools that also run code (such as a [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool) or custom REPL), Claude is operating in a multicomputer environment. The code execution tool runs in Anthropic's sandboxed container, while your client-provided tools run in a separate environment that you control. Claude can sometimes confuse these environments, attempting to use the wrong tool or assuming state is shared between them.

To avoid this, add instructions to your system prompt that clarify the distinction:

This is especially important when combining code execution with [web search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool) or [web fetch](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool), which enable code execution automatically. If your application already provides a client-side shell tool, the automatic code execution creates a second execution environment that Claude needs to distinguish between.

When Claude calls one of your client tools alongside code execution, the API returns the code execution call without its result. The result arrives in a later response, after you send back the `tool_result` blocks for your client tools.

## Streaming

With [streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) enabled (`"stream": true`), you'll receive code execution events as they occur. The sub-tool input streams as `input_json_delta` events, and each result block arrives whole in a single `content_block_start` event:

## Batch requests

You can include the code execution tool in the [Messages Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing). Code execution tool calls through the Messages Batches API are priced the same as those in regular Messages API requests.

## Usage and pricing

**Code execution is free when used with web search or web fetch.** When `web_search_20260209` (or later) or `web_fetch_20260209` (or later) is included in your API request, there are no additional charges for code execution tool calls beyond the standard input and output token costs.

When used without these tools, code execution is billed by execution time, tracked separately from token usage:

*   Execution time has a minimum of 5 minutes
*   Each organization receives **1,550 free hours** of usage per month
*   Additional usage beyond 1,550 hours is billed at **$0.05 USD per hour, per container**
*   If files are included in the request, execution time is billed even if the tool is not called, because files are preloaded onto the container

Code execution usage is tracked in the response:

## Upgrade to latest tool version

The latest tool version is `code_execution_20260521`. To move between the three current versions, update the `type` string in your request: all three return the response blocks documented in [Response format](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#response-format). See [Tool versions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#tool-versions) for what each version adds and [Compatibility](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compatibility) for the models that support them.

The rest of this section covers migrating from the legacy Python-only `code_execution_20250522` to the current tool versions.

### What's changed

| Component | Legacy | Current |
| --- | --- | --- |
| Beta header | `code-execution-2025-05-22` | None required |
| Tool type | `code_execution_20250522` | `code_execution_20250825` or later |
| Capabilities | Python only | Bash commands, file operations |
| Response types | `code_execution_result` | `bash_code_execution_result`, `text_editor_code_execution_*_result` |

### Backward compatibility

*   All existing Python code execution continues to work exactly as before
*   No changes required to existing Python-only workflows

### Upgrade steps

To upgrade, update the tool type in your API requests:

**Review response handling** (if parsing responses programmatically):

*   The API no longer sends the previous blocks for Python execution responses
*   Instead, the API sends new response types for Bash and file operations (see [Response format](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#response-format))

## Data retention

Code execution runs in server-side sandbox containers. Container data, including execution artifacts, uploaded files, and outputs, is retained for up to 30 days. This retention applies to all data processed within the container environment. Files that code execution creates in the [Files API](https://platform.claude.com/docs/en/build-with-claude/files) (retrievable with `client.files.download()`) persist until explicitly deleted.

For ZDR eligibility across all features, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

## Next steps

Pair a faster executor model with a higher-intelligence advisor model that provides strategic guidance mid-generation.

Call your own tools from code that runs inside the code execution container.

Upload files for analysis and download the files that code execution creates.

Learn how to use Agent Skills to extend Claude's capabilities through the API.

## Compatibility

| Supported models | * Fable 5 and 5.1 * Mythos 5 and 5.1 * Opus 4.5, 4.6, 4.7, 4.8, and 5 * Sonnet 4.5, 4.6, and 5 * Haiku 4.5 |
| --- |
| Supported platforms | * Claude API * Claude Platform on AWS * Microsoft Foundry[1](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compat-fn-1) |

1.   On [Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry), code execution requires a [Hosted on Anthropic deployment](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry#additional-features-not-supported-when-hosted-on-azure).[↩](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compat-fnref-1)

*   Every supported model accepts all three [tool versions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#tool-versions). On Claude Haiku 4.5, programmatic tool calling and REPL state persistence aren't available, so the newer versions behave like `code_execution_20250825` there.
*   For [Claude Mythos Preview](https://anthropic.com/glasswing), code execution is supported on the Claude API and Microsoft Foundry.
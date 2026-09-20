---
title: 在会话中上传并挂载文件供 Agent 读取处理
url: https://platform.claude.com/docs/en/managed-agents/files
source_type: web
folder: claude/agent
author: null
tags:
- Claude API
- Files API
- Agent sandbox
- 文件挂载
- Managed Agents
summary: 通过 Files API 上传文件并在会话 resources 中挂载到沙盒，让 Agent 像读本地文件一样处理 CSV、文档和二进制等数据。
fetched_at: '2026-09-20T07:44:34.078928+00:00'
---

Upload files and mount them in your sandbox for reading and processing.

You can provide files to your agent by uploading them through the Files API and mounting them in the session's sandbox.

## Uploading files

First, upload a file using the [Files API](https://platform.claude.com/docs/en/build-with-claude/files):

## Mounting files in a session

Mount uploaded files into the sandbox by adding them to the `resources` array when creating a session:

With the preceding `mount_path`, the agent reads the file at `/mnt/session/uploads/data.csv` (see [File paths](https://platform.claude.com/docs/en/managed-agents/files#file-paths)).

A new `file_id` is created that references the instance of the file in the session. These copies do not count against your [storage limits](https://platform.claude.com/docs/en/build-with-claude/files).

## Multiple files

Mount multiple files by adding entries to the `resources` array:

A maximum of 500 files is supported per session.

## Managing files on a running session

You can add or remove files from a session after creation using the session resources API. Each resource has an `id` returned when it is added (or listed), which you use for deletes.

List all resources on a session with `resources.list`. To remove a file, call `resources.delete` with the resource ID:

## Listing and downloading session files

Use the [Files API](https://platform.claude.com/docs/en/build-with-claude/files) to list files scoped to a session and download them. Files the agent writes to `/mnt/session/outputs/` appear in the list shortly after the agent finishes writing them, sometimes a few seconds after the session goes idle. If an output file you expect is missing, list again after a short delay; once it appears in the list, its upload has finished.

Filtering by `scope_id` requires the `managed-agents-2026-04-01` beta header, so the list examples use the `beta` files namespace and pass that header explicitly.

## Supported file types

The agent can work with any file type, including:

*   Source code (`.py`, `.js`, `.ts`, `.go`, `.rs`, and others)
*   Data files (`.csv`, `.json`, `.xml`, `.yaml`)
*   Documents (`.txt`, `.md`)
*   Archives (`.zip`, `.tar.gz`) - the agent can extract these using bash
*   Binary files - the agent can process these with appropriate tools

## File paths

*   The path you specify is rooted under the session's uploads directory: a `mount_path` of `/data.csv` places the file at `/mnt/session/uploads/data.csv` in the sandbox
*   If you omit `mount_path`, the file is placed at `/mnt/session/uploads/<file_id>`
*   Parent directories are created automatically
*   Paths should be absolute (starting with `/`)
*   Files the agent writes to `/mnt/session/outputs/` become available through the Files API, scoped to the session; see [Listing and downloading session files](https://platform.claude.com/docs/en/managed-agents/files#listing-and-downloading-session-files)
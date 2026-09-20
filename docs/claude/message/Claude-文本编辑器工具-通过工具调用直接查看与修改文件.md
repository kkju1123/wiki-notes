---
title: Claude 文本编辑器工具：通过工具调用直接查看与修改文件
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool
source_type: web
folder: claude/message
author: null
tags:
- Claude
- 工具使用
- 文本编辑器
- Agent
- 文件操作
summary: Claude 文本编辑器工具：讲解 view、str_replace、create、insert 四类命令及安全实现要点。
fetched_at: '2026-09-20T03:59:52.792235+00:00'
---

Give Claude the Anthropic-defined text editor tool to view, create, and edit files, and handle its view, str_replace, create, and insert commands.

Claude can use an Anthropic-schema text editor tool to view and modify text files, helping you debug, fix, and improve your code or other text documents. This allows Claude to directly interact with your files, providing hands-on assistance rather than just suggesting changes.

For model support, see the [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference).

## When to use the text editor tool

Some examples of when to use the text editor tool are:

*   **Code debugging:** Have Claude identify and fix bugs in your code, from syntax errors to logic issues.
*   **Code refactoring:** Let Claude improve your code structure, readability, and performance through targeted edits.
*   **Documentation generation:** Ask Claude to add docstrings, comments, or README files to your code base.
*   **Test creation:** Have Claude create unit tests for your code based on its analysis of the implementation.

## Use the text editor tool

Provide the text editor tool (named `str_replace_based_edit_tool`) to Claude using the Messages API.

You can optionally specify a `max_characters` parameter to control truncation when viewing large files.

Use the text editor tool in the following way:

1.   
### Provide Claude with the text editor tool and a user prompt

    *   Include the text editor tool in your API request
    *   Provide a user prompt that may require examining or modifying files, such as "Can you fix the syntax error in my code?"

2.   
### Claude uses the tool to examine files or directories

    *   Claude assesses what it needs to look at and uses the `view` command to examine file contents or list directory contents
    *   The API response will contain a `tool_use` content block with the `view` command

3.   
### Execute the view command and return results

    *   Extract the file or directory path from Claude's tool use request
    *   Read the file's contents or list the directory contents
    *   If a `max_characters` parameter was specified in the tool configuration, truncate the file contents to that length
    *   Return the results to Claude by continuing the conversation with a new `user` message containing a `tool_result` content block

4.   
### Claude uses the tool to modify files

    *   After examining the file or directory, Claude may use a command such as `str_replace` to make changes or `insert` to add text at a specific line number.
    *   If Claude uses the `str_replace` command, Claude constructs a properly formatted tool use request with the old text and new text to replace it with

5.   
### Execute the edit and return results

    *   Extract the file path, old text, and new text from Claude's tool use request
    *   Perform the text replacement in the file
    *   Return the results to Claude

6.   
### Claude provides its analysis and explanation

    *   After examining and possibly editing the files, Claude provides a complete explanation of what it found and what changes it made

### Text editor tool commands

The text editor tool supports several commands for viewing and modifying files:

#### view

The `view` command allows Claude to examine the contents of a file or list the contents of a directory. It can read the entire file or a specific range of lines.

Parameters:

*   `command`: Must be "view"
*   `path`: The path to the file or directory to view
*   `view_range` (optional): An array of two integers specifying the start and end line numbers to view. Line numbers are 1-indexed, and -1 for the end line means read to the end of the file. This parameter only applies when viewing files, not directories.

#### str_replace

The `str_replace` command allows Claude to replace a specific string in a file with a new string. This is used for making precise edits.

Parameters:

*   `command`: Must be "str_replace"
*   `path`: The path to the file to modify
*   `old_str`: The text to replace (must match exactly, including whitespace and indentation)
*   `new_str`: The new text to insert in place of the old text

#### create

The `create` command allows Claude to create a new file with specified content.

Parameters:

*   `command`: Must be "create"
*   `path`: The path where the new file should be created
*   `file_text`: The content to write to the new file

#### insert

The `insert` command allows Claude to insert text at a specific location in a file.

Parameters:

*   `command`: Must be "insert"
*   `path`: The path to the file to modify
*   `insert_line`: The line number after which to insert the text (0 for beginning of file)
*   `insert_text`: The text to insert

### Example: Fixing a syntax error with the text editor tool

This example demonstrates how Claude uses the text editor tool to fix a syntax error in a Python file.

First, your application provides Claude with the text editor tool and a prompt to fix a syntax error:

Claude uses the text editor tool first to view the file:

Your application should then read the file and return its contents to Claude:

Claude identifies the syntax error and uses the `str_replace` command to fix it:

Your application should then make the edit and return the result:

Finally, Claude provides a complete explanation of the fix:

## Implement the text editor tool

The text editor tool is implemented as a schema-less tool. When using this tool, you don't need to provide an input schema as with other tools; the schema is built into Claude's model and can't be modified.

The tool type is `type: "text_editor_20250728"` for Claude 4 and later models.

1.   ### Initialize your editor implementation

Create helper functions to handle file operations like reading, writing, and modifying files. Consider implementing backup functionality to recover from mistakes. 
2.   ### Handle editor tool calls

Create a function that processes tool calls from Claude based on the command type: 
3.   
### Implement security measures

Add validation and security checks:

    *   Validate file paths to prevent directory traversal
    *   Create backups before making changes
    *   Handle errors gracefully
    *   Implement permissions checks

4.   ### Process Claude's responses

Extract and handle tool calls from Claude's responses: 

### Handle errors

When using the text editor tool, various errors may occur. Here is guidance on how to handle them:

### Follow implementation best practices

* * *

## Pricing and token usage

The text editor tool uses the same pricing structure as other tools used with Claude. It follows the standard input and output token pricing based on the Claude model you're using.

In addition to the base tokens, the following additional input tokens are needed for the text editor tool:

| Tool | Additional input tokens |
| --- | --- |
| `text_editor_20250429` (Claude 4.x) | 700 tokens |

For more detailed information about tool pricing, see [Tool use pricing](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#pricing).

## Integrate the text editor tool with other tools

You can use the text editor tool alongside other Claude tools. When combining tools, ensure you:

*   Match the tool version with the model you're using
*   Account for the additional token usage for all tools included in your request

## Change log

| Date | Version | Changes |
| --- | --- | --- |
| July 28, 2025 | `text_editor_20250728` | Release of an updated text editor tool that fixes some issues and adds an optional `max_characters` parameter. It is otherwise identical to `text_editor_20250429`. |
| April 29, 2025 | `text_editor_20250429` | Release of the text editor tool for Claude 4. This version removes the `undo_edit` command but maintains all other capabilities. The tool name has been updated to reflect its str_replace-based architecture. |
| March 13, 2025 | `text_editor_20250124` | Introduction of standalone text editor tool documentation. This version is optimized for Claude Sonnet 3.7 but has identical capabilities to the previous version. |
| October 22, 2024 | `text_editor_20241022` | Initial release of the text editor tool with Claude Sonnet 3.5 (retired; see [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)). Provides capabilities for viewing, creating, and editing files through the `view`, `create`, `str_replace`, `insert`, and `undo_edit` commands. |

## Next steps

Here are some ideas for how to use the text editor tool in more convenient and powerful ways:

*   **Integrate with your development workflow**: Build the text editor tool into your development tools or IDE
*   **Create a code review system**: Have Claude review your code and make improvements
*   **Build a debugging assistant**: Create a system where Claude can help you diagnose and fix issues in your code
*   **Implement file format conversion**: Let Claude help you convert files from one format to another
*   **Automate documentation**: Set up workflows for Claude to automatically document your code

The text editor tool enables Claude to work directly with your code base, supporting workflows from debugging to automated documentation.

Learn how to implement tool workflows for use with Claude.

Execute shell commands with Claude.
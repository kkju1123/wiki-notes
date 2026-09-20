---
title: Claude PDF 支持：文本抽取与视觉理解
url: https://platform.claude.com/docs/en/build-with-claude/pdf-support
source_type: web
folder: claude
author: null
tags:
- Claude API
- PDF处理
- 多模态
- 文档理解
- Amazon Bedrock
summary: 介绍用 Claude 处理 PDF 的输入方式、工作原理、成本估算与优化实践，覆盖文本和图表视觉分析。
fetched_at: '2026-09-20T03:14:41.584442+00:00'
---

Process PDFs with Claude: extract text, analyze charts, and understand visual content from your documents.

You can ask Claude about any text, pictures, charts, and tables in PDFs you provide. Some sample use cases:

*   Analyzing financial reports and understanding charts/tables
*   Extracting key information from legal documents
*   Assisting with document translation
*   Converting document information into structured formats

## Before you begin

### Check PDF requirements

Claude works with any standard PDF. Ensure your request size meets these requirements:

| Requirement | Limit |
| --- | --- |
| Maximum request size | 32 MB ([varies by platform](https://platform.claude.com/docs/en/api/overview#request-size-limits)) |
| Maximum pages per request | 600 (100 when the request's context window is under 1M tokens) |
| Format | Standard PDF (no passwords/encryption) |

Both limits are on the entire request payload, including any other content sent alongside PDFs. For large PDFs, consider uploading with the [Files API](https://platform.claude.com/docs/en/build-with-claude/files) and referencing by `file_id` to keep request payloads small.

Because PDF support relies on Claude's vision capabilities, it is subject to the same [limitations and considerations](https://platform.claude.com/docs/en/build-with-claude/vision#limitations) as other vision tasks.

### Supported platforms and models

All [active models](https://platform.claude.com/docs/en/models/overview) support PDF processing. For PDF support through Amazon Bedrock's Converse API, see [Amazon Bedrock PDF support](https://platform.claude.com/docs/en/build-with-claude/pdf-support#amazon-bedrock-pdf-support).

### Amazon Bedrock PDF support

When using PDF support through the Converse API, part of [Claude on Amazon Bedrock (Opus 4.6 and earlier)](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy), there are two distinct document processing modes:

#### Document processing modes

1.   **Converse Document Chat** (Original mode - Text extraction only)

    *   Provides basic text extraction from PDFs
    *   Cannot analyze images, charts, or visual layouts within PDFs
    *   Uses approximately 1,000 tokens for a 3-page PDF
    *   Automatically used when citations are not enabled

2.   **Claude PDF Chat** (New mode - Full visual understanding)

    *   Provides complete visual analysis of PDFs
    *   Can understand and analyze charts, graphs, images, and visual layouts
    *   Processes each page as both text and image for comprehensive understanding
    *   Uses approximately 7,000 tokens for a 3-page PDF
    *   **Requires citations to be enabled** in the Converse API

#### Key limitations

*   **Converse API:** Visual PDF analysis requires citations to be enabled. There is currently no option to use visual analysis without citations (unlike the InvokeModel API).
*   **InvokeModel API:** Provides full control over PDF processing without forced citations.

#### Common issues

If Claude isn't seeing images or charts in your PDFs when using the Converse API, you likely need to enable the citations flag. Without it, Converse falls back to basic text extraction only.

## Process PDFs with Claude

### Send your first PDF request

Start with a simple example using the Messages API. You can provide PDFs to Claude in three ways:

1.   As a URL reference to a PDF hosted online
2.   As a base64-encoded PDF in `document` content blocks
3.   By a `file_id` from the [Files API](https://platform.claude.com/docs/en/build-with-claude/files)

#### Option 1: URL-based PDF document

The simplest approach is to reference a PDF directly from a URL:

The response returns Claude's analysis as text blocks in `content`, with token consumption in `usage`:

#### Option 2: Base64-encoded PDF document

If you need to send PDFs from your local system or when a URL isn't available:

#### Option 3: Files API

For PDFs you'll use repeatedly, or when you want to avoid encoding overhead, use the [Files API](https://platform.claude.com/docs/en/build-with-claude/files):

### How PDF support works

When you send a PDF to Claude, the following steps occur:

1.   
### The system extracts the contents of the document.

    *   The system converts each page of the document into an image.
    *   The text from each page is extracted and provided alongside each page's image.

2.   
### Claude analyzes both the text and images to better understand the document.

    *   Documents are provided as a combination of text and images for analysis.
    *   This allows users to ask for insights on visual elements of a PDF, such as charts, diagrams, and other non-textual content.

3.   
### Claude responds, referencing the PDF's contents if relevant.

Claude can reference both textual and visual content when it responds. You can further improve performance by integrating PDF support with:

    *   [Use prompt caching](https://platform.claude.com/docs/en/build-with-claude/pdf-support#use-prompt-caching): To improve performance for repeated analysis.
    *   [Process document batches](https://platform.claude.com/docs/en/build-with-claude/pdf-support#process-document-batches): For high-volume document processing.
    *   [Tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview): To extract specific information from documents for use as tool inputs.

### Estimate your costs

The token count of a PDF file depends on the total text extracted from the document and the number of pages:

*   Text token costs: Each page typically uses 1,500–3,000 tokens per page depending on content density. Standard API pricing applies with no additional PDF fees.
*   Image token costs: Because each page is converted into an image, the same [image-based cost calculations](https://platform.claude.com/docs/en/build-with-claude/vision#evaluate-image-size) are applied.

You can use [token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) to estimate costs for your specific PDFs.

## Optimize PDF processing

### Improve performance

Follow these best practices for optimal results:

*   Place PDFs before text in your requests
*   Use standard fonts
*   Ensure text is clear and legible
*   Rotate pages to proper upright orientation
*   Use logical page numbers (from PDF viewer) in prompts
*   Split large PDFs into chunks when needed
*   Enable prompt caching for repeated analysis

### Scale your implementation

For high-volume processing, consider these approaches:

#### Use prompt caching

Cache PDFs with [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) to improve performance on repeated queries:

#### Process document batches

Use the [Message Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing) to process many PDFs in one request:

Batches process asynchronously. To check progress and retrieve results once processing ends, see [Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing).

## Next steps

Claude's vision capabilities allow it to understand and analyze images, opening up exciting possibilities for multimodal interaction.

Explore practical examples of PDF processing in the Claude Cookbook recipe.

See complete API documentation for PDF support.

## Compatibility

| Supported platforms | * Claude API * Claude Platform on AWS * Amazon Bedrock * Google Cloud * Microsoft Foundry |
| --- |
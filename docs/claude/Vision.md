---
title: Vision
url: https://platform.claude.com/docs/en/build-with-claude/vision
source_type: web
folder: claude
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T03:16:49.972402+00:00'
---

Claude's vision capabilities allow it to understand and analyze images, opening up exciting possibilities for multimodal interaction.

This guide describes how to send images to Claude, the limits and costs that apply, and where to find guidance for [coordinate-based workflows](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates).

* * *

## Send images to Claude

Use Claude's vision capabilities through:

*   [claude.ai](https://claude.ai/). Upload an image like you would a file, or drag and drop an image directly into the chat window.
*   [Playground](https://platform.claude.com/playground) in the Claude Console. Add images directly to any User message block.
*   API request. See the following examples.

On the API, provide images to Claude as `image` content blocks using one of three source types:

1.   A base64-encoded image embedded in the request body
2.   A URL reference to an image hosted online
3.   A `file_id` returned by the [Files API](https://platform.claude.com/docs/en/build-with-claude/files) (upload once, reference many times)

### Base64-encoded image example

### URL-based image example

### Files API image example

For images you'll use repeatedly or when you want to avoid encoding overhead, use the [Files API](https://platform.claude.com/docs/en/build-with-claude/files). Upload the image once, then reference the returned `file_id` in subsequent messages instead of resending base64 data.

See [Messages API examples](https://platform.claude.com/docs/en/api/messages/create) for more example code and parameter details.

### Multiple images

You can include multiple images in a single request, and Claude analyzes them jointly. This is useful for comparing images, asking about differences, or working with a sequence such as pages of a document. When sending several images, introduce each one with a short text label (`Image 1:`, `Image 2:`, and so on) so you can refer to them by name in your prompt and in follow-up turns.

In a multi-turn conversation, add new images in later `user` turns the same way. Claude has access to every image from earlier turns, so follow-up questions such as "Are these similar to the first two?" work without including the earlier images again in the new turn's content.

* * *

## Image limits and costs

### Request limits

The maximum number of images per message or request is:

*   20 per message on [claude.ai](https://claude.ai/).
*   100 per request on the API, for models with a 200k-token context window.
*   600 per request on the API, for all other models.

The maximum dimensions per image are 8000x8000 px.

If a single API request contains more than 20 images, a stricter per-image dimension limit applies to every image in that request. All `image` blocks in the request count toward this threshold, including images from earlier conversation turns that you resend and images nested inside `tool_result` content (for example, screenshots returned to the computer use tool). On Amazon Bedrock and Google Cloud, document blocks such as PDFs also count toward this threshold. Images exceeding the stricter limit are rejected with an `invalid_request_error` whose message references "many-image requests" and states the current limit in pixels. To stay under the limit on all platforms, either resize each image so that neither dimension exceeds 2000 px, or keep the request to 20 or fewer image and document blocks.

The maximum size per image is:

*   10 MB (base64-encoded) when using the Claude API directly.
*   5 MB (base64-encoded) on Amazon Bedrock and Google Cloud.
*   10 MB on [claude.ai](https://claude.ai/).

### Supported formats

Claude supports JPEG, PNG, GIF, and WebP images (`image/jpeg`, `image/png`, `image/gif`, `image/webp`). Animations are unsupported, and only the first frame is used.

### Resolution and token cost

Claude views images in patches instead of pixels. Each patch is a 28×28-pixel block of the image, referred to as a visual token. An image, therefore, costs `⌈width / 28⌉ × ⌈height / 28⌉` visual tokens.

Each model has a maximum native image resolution, expressed as a long-edge limit and a visual-token limit. Images larger than either limit are downscaled before processing; see [How Claude resizes and pads images](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images) for the exact rule. The exception is screenshots and zoom images that you return to the [computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool#handle-coordinate-scaling-for-higher-resolutions) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool#targets-and-coordinates) toolsets: the API rejects a `tool_result` image that exceeds the model's limits with a validation error instead of downscaling it, so resize those images in your application before returning them. To have any other oversized image rejected with an error instead of downscaled, set the image block's [`transformations` field](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#oversized-image-error).

| Resolution tier | Models | Max long edge | Max visual tokens |
| --- | --- | --- | --- |
| High-resolution | Claude 4.7 and later models | 2576 px | 4784 |
| Standard | All other models | 1568 px | 1568 |

High-resolution support is automatic on the listed models and requires no beta header or client-side opt-in.

The following table shows the downsized resolution and visual-token cost for several image sizes on each tier:

| Image size | Standard tier: downsized to | Standard tier: tokens | High-resolution tier: downsized to | High-resolution tier: tokens |
| --- | --- | --- | --- | --- |
| 200x200 px (0.04 megapixels) | Not resized | 64 | Not resized | 64 |
| 1000x1000 px (1 megapixel) | Not resized | 1296 | Not resized | 1296 |
| 1092x1092 px (1.19 megapixels) | Not resized | 1521 | Not resized | 1521 |
| 1920x1080 px (2.07 megapixels) | 1456x819 px | 1560 | Not resized | 2691 |
| 2000x1500 px (3 megapixels) | 1269x952 px | 1564 | Not resized | 3888 |
| 3840x2160 px (8.29 megapixels) | 1456x819 px | 1560 | 2576x1449 px | 4784 |

When an image is downsized, Claude scales it to the largest size that fits the tier's limits while preserving its aspect ratio. This caps the token cost. For the precise rule and a reference implementation, see [How Claude resizes and pads images](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images).

To estimate cost, multiply the token count by the [per-token price of the model](https://claude.com/pricing) you're using. For example, at Claude Haiku 4.5's $1 USD per million input tokens (standard tier), the 1000×1000 image costs about $1.30 USD per thousand images. At Claude Opus 5's $5 USD per million (high-resolution tier), the same image costs about $6.48 USD per thousand and the 4K image about $23.92 USD per thousand.

High-resolution images can use up to roughly three times more visual tokens than the same image on a standard-tier model. If you don't need the additional fidelity that high resolution provides for computer use, screenshot understanding, and dense documents, downsample images before sending to control token costs. To minimize latency and to simplify [coordinate-based workflows](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates), prefer resizing images before uploading them.

### Image quality guidance

When providing images to Claude, keep the following in mind for best results:

*   **Image clarity:** Ensure images are clear and not too blurry or pixelated.
*   **Text:** If the image contains important text, make sure it's legible and not too small. Avoid cropping out key visual context solely to enlarge the text.
*   **Resizing:** Take into account that your image might be resized if it is too large (see [Resolution and token cost](https://platform.claude.com/docs/en/build-with-claude/vision#evaluate-image-size)); this might, for example, make text less legible. Consider pre-resizing your images, cropping them, or both. To have an oversized image rejected with an error instead of resized (important for [coordinate workflows](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates)), mark the image block with [`"oversized_image": "error"`](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates#oversized-image-error).
*   **Image compression:** Compressing images before sending them, using a lossy format such as JPEG or WebP (lossy mode), can reduce latency by reducing the size of requests. However, this can introduce artifacts that are detrimental to model performance, especially when multiple compression passes are applied. For example, heavy JPEG compression can make text difficult to read. Confirm your compression settings are appropriate for the task by inspecting the actual images sent to the API.

* * *

## Coordinates and bounding boxes

For bounding boxes, points, and pixel coordinates, see [Coordinates and bounding boxes](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates). Claude returns absolute pixel coordinates relative to the image it sees after resizing; that guide covers how Claude resizes and pads images and how to pre-resize or rescale so coordinates line up with your original image.

* * *

## Limitations

Although Claude's image understanding capabilities are cutting-edge, there are some limitations to be aware of:

*   **People identification:** Claude [cannot be used](https://www.anthropic.com/legal/aup) to name people in images and refuses to do so.
*   **Accuracy:** Claude might hallucinate or make mistakes when interpreting low-quality, rotated, or very small images under 200 pixels.
*   **Spatial reasoning:** Claude's coordinate and localization outputs are approximate. Follow the guidance in [Coordinates and bounding boxes](https://platform.claude.com/docs/en/build-with-claude/vision-coordinates) and verify outputs before relying on them.
*   **Counting:** Claude can give approximate counts of objects in an image but might not always be precisely accurate, especially with large numbers of small objects.
*   **AI-generated images:** Claude cannot determine whether an image is AI-generated and might be incorrect if asked. Do not rely on it to detect fake or synthetic images.
*   **Inappropriate content:** Claude does not process inappropriate or explicit images that violate the [Acceptable Use Policy](https://www.anthropic.com/legal/aup).
*   **Healthcare applications:** Although Claude can analyze general medical images, it is not designed to interpret complex diagnostic scans such as CTs or MRIs. Claude's outputs should not be considered a substitute for professional medical advice or diagnosis.

Always carefully review and verify Claude's image interpretations, especially for high-stakes use cases. Do not use Claude for tasks requiring perfect precision or sensitive image analysis without human oversight.

* * *

## FAQ

* * *

## Next steps

Get tips and best-practice techniques for tasks such as interpreting charts and extracting content from forms.

See the Messages API documentation, including example API calls involving images.
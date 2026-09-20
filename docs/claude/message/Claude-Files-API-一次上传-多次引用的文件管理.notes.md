# Claude Files API：一次上传、多次引用的文件管理

*原文: [https://platform.claude.com/docs/en/build-with-claude/files](https://platform.claude.com/docs/en/build-with-claude/files) · 来源: web · 生成时间: 2026-09-20T03:12:02.469746+00:00*

## 背景

大模型 API 早期通常把文件内容 base64 编码后直接放进请求体，重复调用时每次都要重传，既消耗带宽又容易撞上请求体积限制。引入代码执行工具后，模型需要读取数据集并产出图表、报告等文件，也需要一个稳定入口来上传输入和下载输出。Files API 就是把这些文件统一管理起来，让请求里只传一个 file_id 指针。

## 痛点

没有 Files API 时，大量或大文件每次请求都要重新编码和传输，延迟高、成本大，且可能超过请求体上限。代码执行生成的产物没有统一下载和管理方式，容易被临时处理逻辑散落各处。

## 解决办法

上传文件到 Anthropic 的 workspace 级安全存储，拿到唯一 file_id；后续 Messages 请求只需通过 document、image 或 container_upload 内容块引用这个 file_id。这个过程类似对象存储中先 PUT 拿到 key，之后所有业务请求只传 key 不传原始文件。上传文件被设计为不可下载，只允许下载 skills 或代码执行工具生成的输出文件，平台借此区分输入和输出。配套 list/get/delete 接口管理文件，并通过 expires_in_seconds 自动过期。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

# 1. 上传一次数据集，之后复用 file_id
file = client.files.create(
    file=open("sales.csv", "rb"),
    purpose="code-execution",
)
file_id = file.id

# 2. 在代码执行工具中引用该文件，不需要再次上传
resp = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=2048,
    tools=[{"type": "code_execution_tool"}],
    messages=[{
        "role": "user",
        "content": [
            {"type": "container_upload", "file_id": file_id},
            {"type": "text", "text": "分析 CSV 中的销售趋势，并生成图表。"}
        ]
    }]
)

# 3. 代码执行生成的输出文件可用 Files API 下载
# generated_file_id = ...  # 从响应中的 bash_code_execution_tool_result 提取
# content = client.files.content(generated_file_id)
# open("chart.png", "wb").write(content.read())
```

代码先用 files.create 上传 CSV 并拿到 file_id，这是“创建一次、多次使用”的关键；然后 Messages 请求只通过 container_upload 块引用 file_id，模型代码执行工具就能读取文件内容。下载部分演示生成产物通过 files.content 读取，说明上传文件与生成文件在下载权限上的差别。

## 关键流程

1. 上传文件到 /v1/files，返回 file_id、bytes、created_at、expires_at 等元数据。
2. 在 Messages 请求中用 document/image/container_upload 内容块引用 file_id。
3. 如果是代码执行工具，模型通过容器读取文件并执行分析、生成图表等。
4. 从响应中的 bash_code_execution_tool_result 提取生成文件的 file_id。
5. 用 GET /v1/files/{id}/content 下载生成文件，或用 list/get/delete 管理文件。

## 关键点

- 文件是 workspace 级隔离的，任何同一 workspace 的请求都能引用，因此绝不能接受不可信来源的 file_id，否则可能造成数据混淆或越权。
- 上传的文件 downloadable=false，只有 skills 或代码执行工具生成的输出文件才可下载，这从 API 设计上区分了输入和输出。
- 不同文件类型对应不同内容块：PDF/文本走 document，图片走 image，数据集等走 container_upload，选错模型工具类型会无法读取或处理。
- 单个文件最大 500MB、组织总存储 1TB，上传后可设置 1 小时到 90 天的 expires_at，这直接影响成本、安全和吞吐设计。
- 文件上传后不可修改或重命名，要更新内容必须上传新文件并删除旧文件，适合不可变文件语义。

## 对比与权衡

- 相比把文件 base64 内嵌在每次 Messages 请求中，Files API 在复用性、带宽消耗和请求体限制上更好，但引入了一次上传的额外步骤，且上传后的输入文件不能通过 API 下载，灵活性不如直接内嵌内容。
- 相比自己搭建对象存储并用 URL 传给模型，Files API 与 Claude Workspace、代码执行工具和 Skills 原生集成更好，但可管理性和外部可访问性不如独立对象存储，不适合需要多系统共享存储的场景。

## 自测问题

**问: 为什么上传自己的文件不能通过 Files API 下载？**

设计上把输入文件和输出文件分开。上传文件本身在客户端已经有原始来源，开放下载会增加泄漏风险和带宽成本；而代码执行工具或 Skills 生成的产物需要统一下载通道，所以 downloadable=true。面试时还可以补充：这保证了文件是“引用式输入、产物式输出”，不要把 Files API 当成通用网盘。

**问: 如何保证某个 file_id 不会访问到别的租户文件？**

文件按 workspace 隔离，API 请求只能访问该 workspace 下已上传或生成的文件；同时要警惕 prompt 注入诱导模型接受不可信 file_id，服务端应在执行前校验归属而不是依赖模型判断。

**问: document、image、container_upload 三种引用方式有什么区别？**

document 用于 PDF 和纯文本，由模型直接读文本；image 用于图像，由视觉模型处理；container_upload 专门把文件放进代码执行工具沙箱，适合 CSV、JSON 等数据集，需要程序读取、分析或生成图表。选型取决于内容类型和任务是“阅读理解还是代码处理”。

**问: 文件设置 expires_at 后会发生什么？**

到期后下载文件内容会返回 404，新 Messages 请求在推理前直接失败；元的 metadata 还能读最多 30 天用于排障。expires_at 只能上传时设置且不可修改，适合临时敏感数据处理，减少存储足迹。

**问: 如果用户上传一个 600MB 的数据集，应该怎么处理？**

Files API 单文件限制 500MB，需要先在客户端拆分、采样、压缩或转成适合分析的格式；也可以提前聚合成 CSV/Parquet 后上传，或对于不需要模型直接处理的原始文件，把外部地址传在文本提示中说明。

## 适用场景

- 多次对话需要重复引用同一份 PDF 或长文档，避免每次重新上传。
- 给代码执行工具投喂 CSV、JSON 等数据集，让模型做统计分析和图表生成。
- 批量处理一批图片，例如产品图分类或 OCR，先上传一次，多个请求循环引用。
- 下载由 Skills 或代码执行工具生成的报告、图表、音视频等产物。

## 标签

`Claude API` `Files API` `文件管理` `代码执行工具` `面试`

# 在会话中上传并挂载文件供 Agent 读取处理

*原文: [https://platform.claude.com/docs/en/managed-agents/files](https://platform.claude.com/docs/en/managed-agents/files) · 来源: web · 生成时间: 2026-09-20T07:44:34.078928+00:00*

## 背景

Agent 通常运行在隔离沙盒中，不能直接访问你的本地磁盘或业务系统文件；为了处理真实数据，需要先把文件上传到平台存储，再注入到某个会话。Files API 负责上传和文件生命周期，resources 挂载则把文件映射成沙盒内路径。这套机制类似容器把宿主机文件或 Volume 挂载进容器。

## 痛点

没有这套机制，你只能把数据片段塞进 prompt，受 token、格式和大小限制，无法处理大 CSV、二进制或归档文件。不理解挂载路径和输入/输出目录，Agent 会读不到文件，或写出的结果无处可查。

## 解决办法

先用 Files API 上传文件，获得稳定的 file_id；创建会话时在 resources 数组里为每个文件声明 file_id 和 mount_path，平台会把文件挂载到沙盒的 /mnt/session/uploads 下。这样 Agent 只需按约定路径读取，无需关心文件来源，且会话内的文件副本不计入存储配额。运行中还可以通过 resources.list/resources.delete 动态增删挂载；Agent 写入 /mnt/session/outputs 的文件会回传到 Files API，供后续列出和下载。

## 关键代码示例

```python
# 概念示例：基于 Claude Files API / Sessions API
# 1. 上传文件，得到文件对象 ID
file_id = files.upload('data.csv').id

# 2. 创建会话时挂载文件到沙盒
session = sessions.create(
    model='claude-sonnet-4-20250514',
    resources=[
        {'type': 'file', 'file_id': file_id, 'mount_path': '/data.csv'}
    ],
)
# Agent 在沙盒中实际读取：/mnt/session/uploads/data.csv

# 3. 管理运行中会话的文件资源
resource = session.resources.list()[0]
session.resources.delete(resource.id)
```

第一步把本地文件上传为平台资源，拿到 file_id，这是后续挂载的引用。第二步创建会话时把 file_id 挂到 mount_path，平台自动将路径根目录映射到 /mnt/session/uploads，所以 mount_path=/data.csv 对应沙盒内 /mnt/session/uploads/data.csv。第三步演示运行中动态列出和删除资源，resource_id 与会话挂钩，而不是删除原始上传文件。

## 关键流程

1. 通过 Files API 上传文件，记录返回的 file_id。
2. 创建会话时在 resources 数组中加入 { file_id, mount_path }。
3. Agent 按 /mnt/session/uploads/<mount_path> 读取挂载文件。
4. 运行中可用 resources.list 查看资源，用 resources.delete 删除某个挂载资源。
5. Agent 把结果写入 /mnt/session/outputs/，稍后通过 Files API 按 session scope 列出并下载。

## 关键点

- file_id 是上传文件对象的标识，resource id 是文件在某个会话中的挂载实例标识，两者生命周期不同。
- mount_path 是相对上传根目录的路径，省略时默认落在 /mnt/session/uploads/<file_id>，显式指定更可读。
- 会话内挂载的文件副本不计入存储限制，但一个会话最多支持 500 个文件。
- 支持几乎所有文件类型：源码、CSV/JSON/YAML、MD/TXT、zip/tar.gz 以及二进制；归档需要 Agent 用 bash 主动解压。
- Agent 输出到 /mnt/session/outputs/ 的文件会变成可在 Files API 中列出的会话文件，列出延迟几秒属正常现象。

## 对比与权衡

- 相比直接把文件内容粘贴进 prompt，文件挂载能稳定处理大文件、二进制和归档文件，但需要额外管理 file_id、mount_path 和资源生命周期。
- 相比让 Agent 直接读取外部云存储 URL，Files API 的上传/挂载在平台内部控制传输和可见性，更稳定可控，但文件需要先上传到 Claude 平台。
- 相比在 prompt 里写死数据，挂载文件能支持同一文件跨多个会话复用；但会话内的挂载副本删除后，原始文件仍需要单独管理。

## 自测问题

**问: file_id 和 resource id 有什么区别？**

file_id 是 Files API 上传后得到的文件对象标识；resource id 是文件挂载到某个 session 后生成的资源实例标识。删除 resource 只是解除该会话内的挂载，不会删除原始文件；删除文件对象才影响所有引用。

**问: 传给 mount_path 时为什么不要写 /mnt/session/uploads 前缀？**

平台已把用户指定路径的根固定在沙盒上传目录下。mount_path=/data.csv 实际会变成 /mnt/session/uploads/data.csv；如果自己写完整路径，可能被叠加或违背约定，导致 Agent 找不到文件。

**问: 如果 Agent 输出文件后 Files API 暂时看不到，可能是什么原因？**

输出文件要等 Agent 写完后由平台异步扫描/上传，可能在会话空闲后几秒才出现在列表。可以重试 list；一旦出现在列表中，说明上传已完成。还要确认写入的是 /mnt/session/outputs/，而不是其他临时目录。

**问: 一个会话内文件数量或大小有什么限制？**

原文明确最多 500 个文件；存储配额方面，会话内挂载副本不计入 storage limits。对于大文件，优先考虑挂载而不是塞入 prompt，具体大小限制以 Files API 最新文档为准。

**问: 支持压缩包是什么意思，Agent 会自动解压吗？**

不会自动解压。挂载只负责把 .zip/.tar.gz 放到沙盒路径，Agent 需要主动调用 bash 工具执行 unzip/tar 等命令解压后再读取内容。

## 适用场景

- 需要让 Agent 分析本地 CSV/JSON/YAML 数据时，先上传挂载再让它读取统计或处理。
- 代码审查场景：把项目源码目录或压缩包挂载进会话，让 Agent 在沙盒内检查代码。
- 处理二进制文件或归档：例如上传图片、PDF 或 zip，让 Agent 用工具解压或识别。
- 需要拿到 Agent 处理后的结果文件时，让它写入 /mnt/session/outputs/ 再通过 Files API 下载。

## 标签

`Claude API` `Files API` `Agent sandbox` `文件挂载` `Managed Agents`

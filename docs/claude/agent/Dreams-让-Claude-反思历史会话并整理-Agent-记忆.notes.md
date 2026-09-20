# Dreams：让 Claude 反思历史会话并整理 Agent 记忆

*原文: [https://platform.claude.com/docs/en/managed-agents/dreams](https://platform.claude.com/docs/en/managed-agents/dreams) · 来源: web · 生成时间: 2026-09-20T07:53:06.580334+00:00*

## 背景

在 AI Agent 长期运行中，记忆库通常由各会话追加写入，形成局部增量更新，逐渐积累重复、冲突和过期信息。传统人工整理成本高且容易丢失上下文，而固定规则难以处理语义层面的冗余。Dreams 利用 Claude 的语义理解能力，把记忆库和一批历史会话作为整体进行综合，自动完成记忆治理。

## 痛点

没有 Dreams 时，记忆库可能越来越臃肿，重复和矛盾条目会直接干扰检索质量，导致 Agent 行为不一致或忽视最新偏好。手动逐条清理既费时又无法从大量历史会话中提炼隐含模式。

## 解决办法

Dreams 的核心是一个异步 job：输入一个已有 memory store 和 1-100 个 session，由选定模型运行 dreaming pipeline，产出另一个独立 output memory store。过程中输入 store 永远只读，输出 store 是克隆后的重组版本，因此可以人工审查后决定启用或丢弃。可以把它类比为对笔记做一次“重写整理版”，而不是在原笔记上逐条修改；原笔记本保留，新版不满意可直接扔掉。instructions 用于高层指导，如关注哪类偏好、保留哪些内容，但不能像精确编辑器那样修改指定条目。

## 关键代码示例

```python
import time
import anthropic

client = anthropic.Anthropic()

dream = client.beta.dreams.create(
    model='claude-sonnet-5',
    memory_store_id='ms_abc123',
    session_ids=['sess_1', 'sess_2', 'sess_3'],
    instructions='Focus on coding style preferences and merge duplicates.'
)
print(dream.status)  # pending

while dream.status in ['pending', 'running']:
    time.sleep(30)
    dream = client.beta.dreams.retrieve(dream.id)

output_store_id = dream.outputs[0]['memory_store']['id']
print(output_store_id)
```

代码先通过 beta.dreams.create 创建一个 Dream，显式传入输入记忆库、历史会话列表和可选的 instructions，返回的资源初始状态为 pending。随后轮询状态直到进入终态，体现了异步任务的处理方式。完成后从 outputs 中取出新生成的 memory store id，可直接附加给后续会话使用，这对应了输入 store 不变、输出独立 store 的核心设计。

## 关键流程

1. 创建 dream：指定模型、输入 memory_store_id、1-100 个 session_ids 以及可选 instructions。
2. 轮询 dream 状态，从 pending 到 running，再到 completed、failed 或 canceled。
3. 如果 dream 进入 running，可通过其 session_id 流式观察底层 pipeline 正在读取和写入的内容。
4. completed 后从 outputs[] 获取新 memory store id，并通过 Memory Stores API 或 Console 审查输出内容。
5. 决定是否采用：将输出 store 附加给未来 session，或删除/归档不需要的输出 store。
6. 可选：将已完成的 dream 归档，使其从默认列表中隐藏但仍可按 ID 读取。

## 关键点

- Dreams 通过全局综合而不是局部写入来维护记忆，能发现语义重复、冲突和隐含模式，这是普通追加写入做不到的。
- 输入 memory store 不会被修改，所有结果写入独立输出 store，提供安全回滚和人工审查空间，适合生产环境。
- instructions 只适用于高层综合指导，不适合单条精确修改；后者应直接使用 Memory Stores API。
- 计费与输入 session 数量和长度线性相关，建议从小批量开始验证整理质量，再逐步扩大规模。
- Dreams 是异步任务，必须正确处理 pending、running、completed、failed、canceled 状态，并注意 failed 或 canceled 时输出 store 可能保留部分内容。

## 对比与权衡

- 相比直接使用 Memory Stores API 逐条增删改，Dreams 能自动跨会话提炼和语义去重，但无法精确控制单条修改，且成本更高。
- 相比基于规则或嵌入向量的去重脚本，Dreams 依赖模型语义理解，能处理“意思相同但文本不同”的重复和矛盾，但耗时可能更长，且效果依赖模型质量。

## 自测问题

**问: Dreams 与普通 memory store 写入有什么本质区别？**

普通写入是局部增量，只追加当前会话产生的记忆；Dreams 是异步全局综合，读取已有 store 和多个历史 sessions，输出全新 store，且输入只读。

**问: 如果想修改某一条记忆，应该通过 Dreams 的 instructions 吗？**

不应该。Dreams 是综合过程，不是文本编辑器，对单条指令通常无效果；应直接在输出 store 上用 Memory Stores API 做精准编辑。

**问: Dreams 失败或取消时，输出 store 会怎样？**

failed 或 canceled 时输出 store 保留已写入的部分内容，可供检查；输入 store 不受影响；可通过 Memory Stores API 删除或归档输出 store 来清理。

**问: 如何降低 Dreams 的成本风险并验证效果？**

成本按 token 计且与输入 session 数量、长度线性相关；先选少量代表性 session 和较小模型跑通，查看输出质量后再扩大批次。

**问: 如何观察一个 dream 的执行过程？**

dream 处于 running 时，其 session_id 指向底层 pipeline session，可以流式订阅 events 实时观察读写内容；终态后该 session 会被归档，历史仍可审计。

## 适用场景

- 长期 coding agent 积累大量用户编码风格、项目结构偏好，定期用 Dreams 合并重复规则并去除过时约定。
- 客服或个人助理 agent 跨多个会话收到大量用户偏好，用 Dreams 提炼统一、最新的偏好集合。
- 在将旧记忆库迁移到新项目或新版本前，先生成一个清理后的记忆库，避免旧噪音影响新任务。
- 多个协作者或子 agent 共享记忆时，定期整合不同来源，解决冲突和冗余。

## 标签

`Claude` `Agent Memory` `Memory Management` `AI 记忆治理` `Dreams`

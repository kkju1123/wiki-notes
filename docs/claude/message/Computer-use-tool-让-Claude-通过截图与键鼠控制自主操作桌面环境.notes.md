# Computer use tool：让 Claude 通过截图与键鼠控制自主操作桌面环境

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) · 来源: web · 生成时间: 2026-09-20T04:01:53.960049+00:00*

## 背景

传统 LLM 主要处理文本和结构化 API，无法直接操作图形界面或没有接口的桌面软件。computer use toolset 作为 Anthropic 定义的客户端工具集，让 Claude 通过截图观察屏幕并用鼠标键盘执行动作，解决跨应用、GUI 驱动的自动化问题，常用于 RPA、软件测试和遗留系统自动化。

## 痛点

没有该工具时，开发者只能靠手工操作、脆弱的 UI 选择器脚本或为每个应用单独开发 API 集成，效率低且难维护。模型也缺少视觉反馈，无法判断点击后的屏幕状态，难以闭环执行复杂桌面任务。不理解其执行协议还容易写出不安全或无法收敛的 agent loop。

## 解决办法

将 `{"type": "computer_toolset_20260801"}` 加入 tools 数组，Claude 会返回带 `toolset_name: "computer"` 的 tool_use 块。客户端在自己的容器或虚拟机中按顺序执行 screenshot、left_click、type 等 17 个成员工具，并把结果作为 tool_result 返回。多个 tool_use 可组成 batch action，必须顺序执行，失败时停止后续并返回固定错误文本，让模型重新规划。这个循环就是 agent loop，其中截图返回图像，其他动作返回短文本确认。

## 关键代码示例

```python
def run_agent_loop(prompt):
    messages = [{'role': 'user', 'content': prompt}]
    tools = [{'type': 'computer_toolset_20260801'}]
    while True:
        resp = client.messages.create(model='claude-sonnet-4-20250514',
                                      max_tokens=4096, tools=tools, messages=messages)
        if resp.stop_reason != 'tool_use':
            return resp.content[0].text
        tool_results, stopped = [], False
        for block in resp.content:
            if block.type != 'tool_use':
                continue
            if stopped:
                tool_results.append(make_error_result(block, 'The batch was stopped because a previous action failed.'))
            else:
                ok, output = execute_member(block.name, block.input)
                tool_results.append(make_result(block, output, is_error=not ok))
                stopped = not ok
        messages += [{'role': 'assistant', 'content': resp.content},
                     {'role': 'user', 'content': tool_results}]

```

这段代码是 computer use 的最小 agent loop 实现：先声明消息和工具集，然后循环调用模型。当模型返回 tool_use 时，按顺序执行每个成员工具并生成 tool_result；若某个动作失败，后续动作返回固定的批量停止错误。所有 tool_result 都通过 tool_use_id 与原 tool_use 对应，并携带 toolset_name，满足协议要求。

## 关键流程

1. 在 API 请求的 tools 数组中加入 computer use 工具集，并提供需要桌面交互的用户提示。
2. Claude 评估是否需要在桌面上操作，若需要则返回一个或多个带 toolset_name: computer 的成员 tool_use 块。
3. 应用在自己的容器或虚拟机中按顺序执行每个 tool_use 块对应的成员动作，并返回等量的 tool_result。
4. Claude 分析 tool_result 后判断是否继续操作；重复执行步骤 3 和 4，直到返回最终文本回复，形成 agent loop。

## 关键点

- computer use 是客户端工具集，模型只发出 tool_use 指令，实际鼠标键盘操作由你的应用在受控环境中执行，这让你能审计和限制高风险动作。
- 它与 browser use tool 定位不同：computer use 操作整个桌面和任意应用，browser use 只读写网页内容，后者不需要完整桌面环境，更轻量。
- 安全风险主要来自 prompt injection，模型可能执行网页或图片中的恶意指令；Anthropic 的分类器会扫描返回的截图等结果并引导模型重新确认，但可联系支持关闭该防护。
- batch actions 必须按顺序执行，不能并行；后一个动作通常依赖前一个动作的结果，比如先点击输入框再输入文字。失败后要停止后续动作，并为每个未执行动作返回固定错误文本。
- 实现 computer use 的关键是正确维护 agent loop：每个 tool_use 块都要有对应的 tool_result，通过 tool_use_id 匹配，并携带相同的 toolset_name，否则会被 API 拒绝。

## 对比与权衡

- 相比 browser use tool，computer use 的能力范围更广，能操作任意桌面应用和本地 GUI，但需要完整桌面环境、资源开销更大且 prompt injection 风险更高；browser use 只读/写网页内容，更轻量，适合纯网页任务。
- 相比传统 RPA 脚本（如 Selenium/Playwright），computer use 由 LLM 动态决策，无需为每个 UI 变化重写脚本，灵活性强；但确定性、执行速度和单位任务成本不如脚本方案，不适合高频、固定流程。

## 自测问题

**问: computer use tool 和普通 function calling 有什么区别？**

computer use 是客户端工具集，由应用在本地环境执行，模型只返回工具调用指令；普通 function calling 通常由服务端或应用直接执行一个函数。computer use 还涉及图像返回，模型需要从截图理解屏幕状态，而 function calling 通常是结构化输入输出。

**问: 为什么 batch actions 不能并行执行？**

因为桌面操作有严格的时序依赖，比如先点击输入框再输入文字、先打开菜单再点击选项；并行执行会导致状态不确定和竞态。所以协议要求顺序执行，并在失败时短路，让模型能基于部分成功状态重新规划。

**问: 如何防范 computer use 中的 prompt injection？**

核心是隔离环境和最小权限，不让模型接触敏感数据或执行不可逆操作。Anthropic 还提供了分类器扫描工具返回内容，识别潜在注入并引导模型确认指令来源。高风险场景应保留人工在环，并可选择关闭自动防护，但需自行承担风险。

**问: computer use 和 browser use 怎么选型？**

如果任务完全在浏览器内完成，且页面元素可访问，优先用 browser use，因为更轻量、更快、更安全；如果需要操作本地桌面应用、处理没有 DOM 的界面、或跨多个应用协调动作，则用 computer use。还要考虑是否需要虚拟显示器和额外资源。

**问: 在实现 agent loop 时最容易忽略哪些协议细节？**

一是忘记给每个 tool_use 返回对应的 tool_result，且 tool_use_id 要匹配；二是 tool_result 必须携带相同的 toolset_name；三是 screenshot 和 zoom 需要返回图片，其他动作只需短文本；四是 batch 失败时后续动作要返回固定的停止错误文本，不能只返回部分结果。

## 适用场景

- 自动化桌面 GUI 测试，让 Claude 模拟用户操作并验证界面状态。
- 操作没有 API 的遗留桌面软件，例如自动录入数据、导出报表或点击菜单。
- 跨应用流程自动化，比如从浏览器下载文件、用本地应用处理后再移动到指定文件夹。
- RPA / 数字员工场景，让模型根据自然语言指令完成桌面上的多步骤任务。

## 标签

`Computer Use` `Client Toolset` `Agent Loop` `Prompt Injection` `Desktop Automation`

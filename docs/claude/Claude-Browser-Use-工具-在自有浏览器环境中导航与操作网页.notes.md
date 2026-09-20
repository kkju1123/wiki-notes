# Claude Browser Use 工具：在自有浏览器环境中导航与操作网页

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) · 来源: web · 生成时间: 2026-09-20T04:03:53.270803+00:00*

## 背景

大语言模型要完成真实网页任务，不能只靠文本生成，还需要与页面交互。传统脚本或爬虫常用固定选择器，难以应对 JS 动态渲染和复杂交互。Anthropic 把浏览器能力封装成 client toolset，让 Claude 提出结构化操作意图，由应用在自己的浏览器中执行，既保留模型推理能力，又复用浏览器真实环境。

## 痛点

只用 web fetch 或 search 无法执行点击、登录、填表、上传等操作，也无法稳定读取 JS 单页应用；直接用 computer use 则依赖截图坐标，缺少 DOM 结构信息，定位较脆弱。若不了解该工具的执行循环和批处理失败规则，返回结果时漏掉 tool_use_id 或 toolset_name 会被 API 拒绝。

## 解决办法

它通过一个 browser_toolset_20260801 工具集注册到 Messages API，模型不直接碰浏览器，而是输出 navigate、read_page、left_click 等成员工具调用。页面被抽象为两条互补通道：可访问性树或元素引用提供结构化定位（如点击 ref_2），截图与视口坐标提供视觉信息。应用执行器按顺序运行这些调用，并把每个 tool_use 的结果以 tool_result 返回，模型根据结果继续下一步。批处理动作必须顺序执行，首个失败后后续调用返回固定 HALT 错误文本。可以类比为：模型拿的是遥控器，应用是真正执行动作的手，模型通过仪表盘决定下一轮按哪个按钮。

## 关键代码示例

```python
HALT = 'Not executed: an earlier action in this turn failed.'

def run_browser_batch(tool_use_blocks, handlers):
    results, failed = [], False
    for block in tool_use_blocks:
        tid, ts = block['id'], block['toolset_name']
        if failed:
            results.append({'type':'tool_result','tool_use_id':tid,'toolset_name':ts,'is_error':True,'content':HALT})
            continue
        try:
            results.append({'type':'tool_result','tool_use_id':tid,'toolset_name':ts,'content': handlers[(ts, block['name'])](block['input'])})
        except Exception as exc:
            results.append({'type':'tool_result','tool_use_id':tid,'toolset_name':ts,'is_error':True,'content':str(exc)})
            failed = True
    return results
```

这段代码模拟浏览器批处理执行器：遍历 assistant 响应中的 tool_use 块，用 (toolset_name, name) 做分派键，避免和自定义工具重名；每个结果都带 tool_use_id 和 toolset_name，符合协议。任何 handler 失败后停止执行后续动作，并给它们返回固定 HALT 文本，实现文档要求的 halt rule。

## 关键流程

1. 在 Messages API 请求的 tools 数组中添加一个类型为 browser_toolset_20260801 的条目，并给出需要操作网页的用户提示。
2. 收到 stop_reason 为 tool_use 后，遍历 response.content 中的所有 tool_use 块，按出现顺序执行，不要假设只有一个。
3. 每个块执行后返回一个 tool_result，匹配 tool_use_id 并回显 toolset_name 为 browser；失败块返回 is_error 为 true 和错误描述。
4. 将结果作为新的 user 消息发送，继续循环，直到 Claude 返回普通文本答案。

## 关键点

- browser use 是 client toolset，Claude 只负责决策，应用负责真实浏览器自动化，Anthropic 侧不运行浏览器；这决定了安全、计费与部署责任在应用方。
- 它同时利用可访问性树或元素引用和截图或坐标，前者让点击和读取更稳定，后者补充视觉布局与像素级定位。
- 同一 turn 中出现多个 member calls 属于 batch action，必须顺序执行而非并发，因为后面调用可能依赖前面调用改变页面状态。
- 每个 tool_result 必须对应 tool_use_id 并回显 toolset_name，分派时建议用 (toolset_name, name)，否则遇到同名自定义工具会冲突。
- 页面内容是 untrusted input，执行操作会产生真实副作用，部署前要限制可访问站点、权限、敏感数据处理和人工确认。

## 对比与权衡

- 相比 computer use tool，browser use 能利用 DOM 或可访问性树和元素引用，点击、填表、读取结构更稳定；但只能操作浏览器网页，不能覆盖桌面应用、文件管理器等浏览器外操作。
- 相比 web fetch 或 web search 这类 server tools，browser use 能执行 JS、登录、点击和填表，适合真实操作页面；但更重、延迟更高，需要应用自己运行并维护浏览器，且安全风险更大。

## 自测问题

**问: browser use 和 computer use 的本质区别是什么？**

browser use 面向浏览器网页，提供可访问性树、元素引用、表单、tab 和截图坐标，能精确定位页面元素；computer use 面向整个桌面，只有截图和坐标，适合跨应用操作。选型时看任务是否只在浏览器内，以及是否能利用 DOM 结构。

**问: 为什么同一 turn 多个 tool_use 要顺序执行，还规定失败后后续返回固定文本？**

这些调用通常是连续动作，后续依赖前一步改变的状态；如果某步失败还继续执行，可能产生错误结果或副作用。因此必须停止，并给后续块返回 is_error 为 true 和固定 halt 文本，保证模型理解真实执行状态。

**问: 为什么每个 tool_result 都要回显 toolset_name，并建议按 (toolset_name, name) 分派？**

因为同一请求可能包含自定义工具，名字可能和成员工具相同；回显 toolset_name 标识它属于 browser 工具集，按二元组分派可以避免歧义。

**问: 部署 browser use 时安全上重点考虑什么？**

页面内容不可信，可能注入提示或诱导模型操作；动作可能产生真实世界后果。应限制可访问站点、用隔离浏览器或沙箱、限制表单提交和支付等高风险操作、处理敏感数据，并对关键动作加人工确认或权限控制。

## 适用场景

- 自动化电商后台、客服后台等没有 API 的系统，完成查询、导出、提交工单等操作。
- 对 JS 单页应用进行端到端流程测试或数据抓取，例如点击弹窗、滚动加载、筛选后提取内容。
- 登录后访问需要会话或 Cookie 的页面，抓取个性化数据或执行个人账号操作。
- 研究类任务：在多页面之间跳转、搜索、对比信息，并汇总结果。

## 标签

`Claude API` `Browser Use` `Tool Use` `Agent Loop` `浏览器自动化`

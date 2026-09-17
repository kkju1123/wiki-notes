# Black：不妥协的 Python 代码格式化器

*原文: [https://github.com/psf/black](https://github.com/psf/black) · 来源: github · 生成时间: 2026-09-17T03:11:02.324578+00:00*

## 背景

Python 社区长期依赖 PEP 8 风格指南，但手动执行成本高，代码审查常被空格、换行等格式问题占据。早期工具如 autopep8 只能修补局部违规，无法提供全局一致的排版。Black 由 Łukasz Langa 开发，借鉴 gofmt 的理念，让格式化完全自动化、无争议，聚焦代码逻辑本身。

## 痛点

没有统一格式化工具时，团队会在无关紧要的格式上反复争论，code review 噪音大；手动调整缩进、行宽、引号既耗时又容易引入不一致。大量时间浪费在排版而非业务逻辑上。

## 解决办法

Black 将源码解析为具体语法树（CST），保留注释和所有 token，然后按固定规则重新生成代码，彻底忽略原有格式。格式化后它再比较前后 AST 是否等价，确保语义不变（可用 --fast 跳过）。默认采用 88 字符行宽、双引号、运算符两侧空格、尾随逗号等决策，几乎不提供风格配置，从而实现确定性输出和最小 diff。

## 关键代码示例

```python
# 格式化前
def add(a,b): return a+b
result = [1,2,3]

# 格式化后
def add(a, b):
    return a + b

result = [
    1,
    2,
    3,
]

```

示例展示 Black 如何把紧凑写法拆分为多行，并在逗号后、运算符两侧加空格。长列表被拆成多行并添加尾随逗号，这样后续增删元素时 diff 更小。核心原理是 Black 基于 CST 重新生成代码，而不是局部修补；类似地它还会统一字符串引号、空行等细节。

## 关键流程

1. 安装：pip install black（需要 Python 3.10+）
2. 运行：black {文件或目录} 或 python -m black {文件或目录}
3. 通过 pyproject.toml 配置项目级 include/exclude 等少量选项
4. 在 CI 中用 black --check . 验证格式，非零退出码阻止合并
5. 可选：使用 --fast 跳过 AST 等价性检查以加速（生产环境不推荐）

## 关键点

- Black 是 PEP 8 兼容但高度 opinionated 的格式化器，几乎所有格式决策都已固定，用户无需在风格上做选择。
- 它通过解析为 CST（具体语法树）而不是 AST 来工作，因为 CST 保留注释和原始 token，能完整重新输出代码。
- 格式化前后进行 AST 等价性检查，确保不改变语义，这是它可以安全自动重排的关键保障。
- 默认配置极少，仅暴露少量选项（如 line-length、include/exclude），这种设计有意牺牲灵活性以换取一致性。
- Black 被 pytest、Django、SQLAlchemy、pandas 等大量主流项目采用，已成为 Python 社区事实标准之一。
- 它生成的代码在不同项目中看起来一致，且产生最小 diff，显著降低 code review 的格式噪音。

## 对比与权衡

- 相比 autopep8，Black 在代码一致性和彻底性上更好，会重新排版整个文件；但 autopep8 可以只修复局部 PEP 8 违规，侵入性更小。
- 相比 YAPF，Black 在零配置和社区采用上更好，开箱即用；但 YAPF 允许更细的样式定制，适合有特殊格式要求的团队。
- 相比 ruff format（Rust 实现），Black 在纯 Python 实现、格式兼容参考性和成熟度上更好；但 ruff format 速度更快，适合超大型代码库，且默认风格基本兼容 Black。

## 自测问题

**问: Black 如何保证格式化不会改变代码语义？**

它先解析源码为 CST，按规则重新生成代码，再解析前后两个版本的 AST 进行等效性比较；如果发现不等价会报错并保留原文件。用 --fast 可以跳过该检查以提速。

**问: 为什么 Black 默认行宽是 88 而不是 PEP 8 建议的 79？**

88 是 Black 作者选择的折中，比 79 多约 10%，减少不必要的换行，同时便于在常见屏幕和代码审查工具中并排显示。该值可通过 --line-length 调整，但社区普遍接受默认值。

**问: Black、autopep8、YAPF 三者本质区别是什么？**

autopep8 只针对 PEP 8 违规做局部修补，不会重排整体结构；YAPF 基于格式规则可配置输出风格；Black 是 opinionated，固定绝大多数规则，目标是消除风格决策而非提供灵活性。

**问: 如何把 Black 集成到团队工作流和 CI？**

本地使用 pre-commit 在 git commit 前自动运行；CI 中执行 black --check . 检测格式偏差；统一使用 pyproject.toml 管理项目级配置；大型迁移可用 black --diff 预览变更。

**问: Black 为什么使用 CST 而不是 AST？**

AST 会丢失注释、括号、空行和原始空白等排版信息，无法完整还原代码；CST 保留每个 token 及其位置，允许 Black 在重新输出时保留注释并精确控制格式。

## 适用场景

- 新项目初始化时直接引入 Black，从第一天统一代码风格。
- 既有项目迁移到 Black：一次性全量格式化，提交一个大 diff，之后维护一致性。
- CI 流水线中用 black --check 作为门禁，防止不合规代码合并。
- 在 code review 前自动格式化，减少格式争论，聚焦逻辑和设计。

## 标签

`Python` `Code Formatter` `Black` `PEP 8` `Developer Tools`

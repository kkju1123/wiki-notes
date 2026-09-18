# Jinja：Python 生态中的快速、可扩展模板引擎

*原文: [https://github.com/pallets/jinja](https://github.com/pallets/jinja) · 来源: github · 生成时间: 2026-09-18T06:07:51.131592+00:00*

## 背景

Web 应用需要在服务端生成 HTML、邮件等动态文本。直接拼接字符串会让业务逻辑与展示耦合、难以维护，并容易引发 XSS。Jinja 参考 Django 模板语法，但有意提供更接近 Python 的表达能力，定位为既安全又不束缚模板设计师的通用模板引擎。

## 痛点

没有模板引擎时，HTML 嵌入 Python 代码或用字符串拼接，改样式就要动业务代码，特别容易出错。手写 HTML 转义常被遗漏，用户输入可造成 XSS；同时页面布局重复代码无法复用，模板报错也难以定位到具体行。

## 解决办法

Jinja 在模板中使用 `{{ 表达式 }}` 输出、`{% 语句 %}` 控制流程、`{# 注释 #}` 注释，语法类似 Python 但只保留模板需要的控制能力。模板继承通过 `{% block %}` 让子模板只填充父布局中的可替换区域，宏和 include 复用片段。渲染时 Jinja 先把模板编译成优化的 Python 代码并缓存，随后用传入的上下文变量执行。HTML 模板默认可开启 autoescape，对输出变量做转义；沙箱环境限制属性和调用，能渲染不可信模板。

## 关键代码示例

```python
from jinja2 import Environment, DictLoader

base = '<html><title>{% block title %}Default{% endblock %}</title><main>{% block content %}{% endblock %}</main></html>'

page = '''{% extends 'base.html' %}
{% block title %}Members{% endblock %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href='{{ user.url }}'>{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}'''

env = Environment(loader=DictLoader({'base.html': base, 'page.html': page}), autoescape=True)
print(env.get_template('page.html').render(users=[{'url': '/u/1', 'username': 'Alice'}]))
```

这段代码先创建 Environment 并传入 DictLoader，把 base.html 和 page.html 放到内存字典中，相当于模板加载器。父模板 base 定义了 title 与 content 两个 block 的默认结构；子模板 page 通过 extends 声明继承，再覆盖具体 block。for 循环在渲染时遍历 users 列表，{{ user.url }} 会经过 autoescape 后输出到 HTML，避免用户字段中的特殊字符被当标签执行。

## 关键流程

1. 创建 Environment：配置加载器、autoescape 等选项
2. 编写父模板：用 block 声明可替换区域
3. 编写子模板：extends 继承父模板并覆盖 block
4. 调用 render 传入上下文数据，引擎编译并执行模板

## 关键点

- 模板继承通过 block/extends 把页面骨架与局部内容分离，大幅减少重复布局代码。
- autoescape 默认对 HTML 输出进行转义，是防 XSS 的关键防线；处理可信 HTML 时需显式使用 safe 过滤器。
- Jinja 将模板即时编译为 Python 代码并缓存，后续渲染直接复用，兼顾开发灵活性与运行性能。
- 沙箱环境可以限制不可信模板的属性访问和函数调用，适合用户自定义模板。
- 模板异常会映射回模板中的真实行号，这对快速定位拼写错误和逻辑问题非常重要。
- 过滤器、测试、全局函数和扩展语法让 Jinja 能在不破坏模板简洁性的前提下扩展。

## 对比与权衡

- 相比 Django 模板，Jinja 的表达式语法更接近 Python，允许更灵活的过滤器和函数调用，因此开发者写复杂展示逻辑更顺手；但 Django 模板默认更严格，某些团队可能认为它对非程序员模板作者更友好。
- 相比 Mako，Jinja 对 autoescape 和沙箱的支持更开箱即用，安全性默认更好；但在需要嵌入大量自由 Python 逻辑、追求极致渲染性能的场景下，Mako 可能更灵活。
- 相比直接使用 f-string 或字符串拼接，Jinja 把展示逻辑从业务代码中抽离，继承、include、宏等机制也让 HTML 片段更易复用；但会引入模板语言学习成本和额外渲染开销。

## 自测问题

**问: Jinja 的 autoescape 是如何防止 XSS 的？**

autoescape 会对 `{{ }}` 输出的字符串中的 `&`、`<`、`>`、单引号、双引号等字符做 HTML 实体转义，浏览器会按文本而不是标签解析；需要输出可信 HTML 时要显式使用 `|safe`，但必须确保内容已清洗。

**问: 模板继承中 block 和 extends 的执行顺序是怎样的？**

子模板声明 extends 后，Jinja 先定位父模板，再按父模板的 block 结构渲染；子模板中同名的 block 内容会替换父模板的默认内容。父模板里 block 外的普通内容不能被覆盖，通常放布局骨架。

**问: Jinja 的编译缓存机制是怎样的？**

模板源码首次加载时被解析成 AST，再生成优化后的 Python 代码并编译为可调用对象，按模板名和加载器缓存；默认 auto_reload 会检测文件变更，生产环境可关闭以提高性能。

**问: 如果让用户提交模板，为什么需要沙箱环境？**

沙箱通过限制属性访问、禁止危险内置函数和私有 API 来降低不可信模板读取文件系统、导入模块或执行任意代码的风险；它不能保证绝对安全，还需要配合超时、资源限制和最小权限环境。

**问: 宏和 include 如何选择？**

宏适合带参数的 UI 片段，像函数一样可传入不同参数复用；include 更偏向插入一段完整模板内容且共享当前上下文。宏还能在模板内 import，利于组织可复用组件。

## 适用场景

- Flask 等 Web 框架中的 HTML 页面渲染
- 邮件内容、报表或配置文件等非 HTML 文本生成
- 需要让用户自定义模板但又要限制权限的 SaaS 功能
- 需要复用页面布局、组件和宏的中大型 Web 项目

## 标签

`模板引擎` `Python` `Jinja` `Web开发` `Flask`

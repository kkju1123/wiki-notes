---
title: Click：用最少的代码构建漂亮的 Python 命令行工具
url: https://github.com/pallets/click
source_type: github
author: pallets
tags:
- Python
- CLI
- Click
- 命令行工具
- Pallets
summary: Click 是 Pallets 出品的 Python 命令行接口创建工具包，用装饰器让 CLI 开发变得简单、可组合且自带合理默认值。
fetched_at: '2026-09-16T11:33:28.926854+00:00'
---

<div align="center"><img src="https://raw.githubusercontent.com/pallets/click/refs/heads/stable/docs/_static/click-name.svg" alt="" height="150"></div>

# Click

Click is a Python package for creating beautiful command line interfaces
in a composable way with as little code as necessary. It's the "Command
Line Interface Creation Kit". It's highly configurable but comes with
sensible defaults out of the box.

It aims to make the process of writing command line tools quick and fun
while also preventing any frustration caused by the inability to
implement an intended CLI API.

Click in three points:

-   Arbitrary nesting of commands
-   Automatic help page generation
-   Supports lazy loading of subcommands at runtime


## A Simple Example

```python
import click

@click.command()
@click.option("--count", default=1, help="Number of greetings.")
@click.option("--name", prompt="Your name", help="The person to greet.")
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for _ in range(count):
        click.echo(f"Hello, {name}!")

if __name__ == '__main__':
    hello()
```

```
$ python hello.py --count=3
Your name: Click
Hello, Click!
Hello, Click!
Hello, Click!
```


## Donate

The Pallets organization develops and supports Click and other popular
packages. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today][].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/

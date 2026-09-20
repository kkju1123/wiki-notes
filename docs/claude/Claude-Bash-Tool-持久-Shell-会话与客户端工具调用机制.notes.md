# Claude Bash Tool：持久 Shell 会话与客户端工具调用机制

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool) · 来源: web · 生成时间: 2026-09-20T03:57:49.876758+00:00*

## 背景

在 LLM 工具调用中，Claude 自身不能直接执行命令或访问文件系统，必须通过客户端工具与应用交互。Bash tool 出现的背景是：让模型具备真实执行 shell 命令的能力，从而完成构建、测试、数据处理等操作性任务。由于 Claude API 本身是无状态的，请求之间不会保存 shell 环境，因此需要应用保持一个长期存活的 bash 进程来维持状态。

## 痛点

如果没有 Bash tool，模型只能生成文本建议，无法自动运行命令、创建文件或安装依赖。即使调用普通命令执行，如果每次启动新 shell，cd、export、中间产物都会丢失，多步骤任务无法完成。此外，若直接执行模型生成的任意命令而不做隔离和校验，会带来严重安全风险。

## 解决办法

应用创建并持有一个长期运行的 bash 进程，所有命令通过 stdin 写入，输出通过 stdout/stderr 读取；每条命令后打印一个唯一 sentinel 标记输出结束。Claude 在 tool_use block 中返回 command，应用执行后把 stdout 和 stderr 合并放入 tool_result，再交回 Claude，循环直到模型不再请求工具。可以把整个机制理解为：Claude 是远程用户，应用是终端代理，持久 bash 进程是真正执行命令的 shell。安全上不是靠黑名单拦截，而是靠容器/VM 隔离、白名单校验、资源限制和输出脱敏形成纵深防御。

## 关键代码示例

```python
import subprocess
import uuid

class BashSession:
    def __init__(self):
        self.sentinel = f"__END_{uuid.uuid4().hex}__"
        self.proc = subprocess.Popen(
            ["bash"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            bufsize=1,
        )

    def run(self, command: str) -> str:
        # 写入命令，再用 echo 输出唯一 sentinel 和退出码
        self.proc.stdin.write(f"{command}\n")
        self.proc.stdin.write(f"echo {self.sentinel} $?\n")
        self.proc.stdin.flush()

        output = []
        for line in self.proc.stdout:
            if line.startswith(self.sentinel):
                break
            output.append(line)
        return "".join(output)
```

这个类在初始化时启动一个持久 bash 子进程，stdout 和 stderr 合并以便按发生顺序输出。run 方法把命令写入 stdin，然后追加一句 echo 带唯一 sentinel 和上一条命令退出码；读取输出直到看到 sentinel，从而准确切分每次命令的输出。注意生产实现还需要超时、进程组 kill、输出大小限制和错误处理。

## 关键流程

1. 创建持久 bash 会话：启动一个长期 bash 进程，并为其设置唯一 sentinel 用于标记输出结尾。
2. 解析 Claude 的工具调用：从响应中的 tool_use block 提取 command 和 restart 参数。
3. 在持久会话中执行命令：将 command 写入 bash 的 stdin，并追加 sentinel 输出以确定命令结束。
4. 收集输出：按行读取 stdout/stderr，直到遇见 sentinel 行，把结果作为 tool_result 返回。
5. 循环处理：只要 stop_reason 为 tool_use，就继续执行下一轮工具调用。
6. 补充安全与错误处理：命令失败时返回 is_error=true；超时则杀掉进程组并重启会话，同时对命令做白名单校验和资源限制。

## 关键点

- 持久会话是核心：它让多步骤任务能共享工作目录、环境变量和文件，否则每条命令都在新环境中执行，无法完成链式操作。
- API 无状态与会话有状态分离：Claude API 每次请求不携带 shell 状态，会话生命周期完全由应用控制，这给了集成方灵活性也要求其负责清理。
- 使用 sentinel 标记输出结束：因为管道是持续打开的，必须用唯一标记来切割不同命令的输出，不能依赖 EOF。
- 安全必须纵深防御：白名单校验只能拦截明显错误，真正边界是容器/VM 隔离；还要限制资源、记录审计日志、脱敏输出。
- 错误处理要显式：命令失败时用 is_error=true 返回错误信息，Claude 才能感知失败并修正下一步。
- 版本选择：新集成使用 bash_20250124，不需要 beta header 且支持当前模型；旧版仅兼容已退役模型。

## 对比与权衡

- 相比无状态命令执行（每次启动新 shell），Bash tool 的持久会话能保留工作目录、环境变量和文件，适合多步骤任务，但需要应用管理进程生命周期和超时，复杂度更高。
- 相比固定 schema 的普通 API 工具，Bash tool 是 schema-less 的，模型可以生成任意 shell 命令，灵活性更强，但误用风险也更大，必须配合沙箱隔离。
- 相比托管代码执行环境（如一次性容器），Bash tool 默认由集成方自行控制 shell 进程，更轻量、可嵌入，但隔离性和资源管理需要自己实现。

## 自测问题

**问: 为什么 Bash tool 需要持久会话？API 无状态是什么意思？**

因为 Claude API 每次请求之间不保存任何执行上下文，如果每个命令都新起 shell，前面的 cd、export 和临时文件都会丢失。持久会话由应用维护一个长期 bash 进程，使多轮工具调用共享同一工作目录和环境变量；应用负责会话的创建、存活时间与重启策略。

**问: 如何判断一条命令的输出已经结束？**

由于 bash 进程的 stdout 管道不会 EOF，不能靠读不到数据判断结束。通常做法是在命令后追加 echo 一个唯一 sentinel，并带上退出码；应用逐行读取直到遇见 sentinel。生产实现还要加超时、输出大小限制，防止命令挂起或输出无限大。

**问: 安全上为什么推荐 allowlist 而不是 blocklist？**

blocklist 只能拦截已知危险命令，攻击者可以用空格、编码、命令拼接等方式绕过；allowlist 只允许预期内的命令和操作符，能显著缩小攻击面。但 allowlist 也不是最终边界，模型生成的命令变化多，真正的安全边界是容器/VM 隔离、资源限制、日志审计和输出脱敏。

**问: 如何处理命令超时或挂起？**

需要为每次命令执行设置超时，超时后 kill 整个进程组（包括子进程），然后重启 bash 会话并告知 Claude。不能只 kill bash 本身，因为子进程可能继续运行；Linux 下可用 start_new_session=True 或 os.setsid 创建进程组，再用 os.killpg 终止。

**问: restart 参数的作用是什么？**

当会话状态被污染或需要干净环境时，Claude 可以设置 restart=true。应用应杀掉当前 shell 进程，启动新进程，并返回确认信息。重启后的会话丢失工作目录、环境变量和运行中进程。常用于长任务后的清理，或模型判断当前环境已不可信时。

## 适用场景

- 开发工作流：自动运行构建、测试、lint 和开发工具。
- 系统自动化：执行脚本、管理文件、批量任务。
- 数据处理：处理文件、运行分析脚本、管理数据集。
- 环境配置：安装依赖、配置环境变量、初始化项目。

## 标签

`Claude` `Bash Tool` `tool use` `持久会话` `客户端工具安全`

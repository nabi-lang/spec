# 9.8. Jupyter 内核集成

本节定义 [Jupyter 界面](../000-meta/001-execution-modes.md)的 Nabi 内核。

## 9.8.1. 内核

Jupyter 把 Nabi 内核作为独立进程运行，并通过基于 ZeroMQ 的 Jupyter 线路协议与之通信。内核宣告的 `protocol_version` 为 `"5.3"`（`jupyter_client` 期望的值；适用于 5.3 及更高版本的客户端）。在这种形式下，所有操作系统资源，即文件、网络和外部进程，都可以使用。

内核与终端 REPL 运行在**同一种单元会话**上，并遵循 [9.7.0](907-jit-cells.md#970-单元会话的规则) 的规则：每个单元恰好运行一次，绑定和项延续到之后的单元，失败的单元不在绑定中留下任何痕迹。内核的引擎与 REPL 是同一个 [JIT 会话](906-jit-engine.md)：顶层 `await` 和 `it` 的行为与 REPL 中相同（[9.7.1](907-jit-cells.md#971-单元的种类)、[9.7.7](907-jit-cells.md#977-可视化--输出)）。

### 输入

单元通过客户端读取 stdin。当 `execute_request` 带有 `allow_stdin: true` 时，运行时每次读取 stdin 都会在 stdin 通道上向该请求的客户端发送一个 `input_request`（`prompt` 为空，`password` 为 false）；单元在读取之前写出的内容（例如它打印的提示）已经先作为 `stream` 发送。`input_reply` 的 `value` 加上一个换行，就是这次读取返回的内容。没有 `allow_stdin` 时，或者对于 `silent` 请求，读取会得到输入结束。在单元等待回复时发生的中断会结束该单元（9.8.4）。

### 笔记本单元与 REPL 单元

一个笔记本单元按终端 REPL 拆分输入时所用的相同规则拆分为多个 REPL 单元，并按顺序运行。与 IPython 一样，**只有最后一个值**作为结果显示。如果笔记本单元的后面部分失败，前面部分（之前的 REPL 单元）的效果仍然保留。

REPL 单元在运行时位置（`<repl cell N>`，[9.7.5](907-jit-cells.md#位置)）中的编号取自执行计数：只构成一个 REPL 单元的笔记本单元编号为 `N`，拆分为多个 REPL 单元的笔记本单元把它们编号为 `N.1`、`N.2` 等。不推进计数的请求（`silent`，或 `store_history: false`）把它的 REPL 单元接着当前计数下的最后一个编号继续编号。

## 9.8.2. 本地原生内核：运行方式

当用户选择“Nabi”作为笔记本的内核时，Jupyter 把 kernelspec（9.8.3）中的命令作为独立进程运行。

```
nabi jupyter --connection-file <path>        # resolves dependencies; also accepts build options such as --features
```

内核从连接文件中读取 ZMQ 端口和密钥，然后绑定并监听它的五个套接字。五个套接字都能接受连接后，它在开始捕获输出之前向自己的 stderr 写入一行 `nabi jupyter: ready`。只支持 `tcp` 传输（不支持 `ipc`），不使用 CURVE 加密。消息用连接文件中的密钥进行 HMAC 签名。

内核像 `nabi repl .` 那样（[9.7.10](907-jit-cells.md#9710-加载程序repl-path)）加载它启动时所在的包（当前目录，或其上方最近的含有 `src/main.nabi` 或 `src/lib.nabi` 的目录）：库包作为依赖，并以 `use <name>::*;` 作为第一个单元；程序包则被加载并运行。这发生在开始捕获输出之后；它打印的内容与第一个被执行的单元的输出一起发送。在包之外，会话以空状态开始。

### 套接字通道

| 通道     | 用途                                                    |
|-------------|------------------------------------------------------------|
| `shell`     | 请求（execute、kernel_info、complete、is_complete 等） |
| `iopub`     | 输出和状态广播（status、stream、execute_result 等） |
| `stdin`     | 供读取 stdin 的单元使用的 `input_request` / `input_reply`（9.8.1） |
| `control`   | interrupt、shutdown                                        |
| `heartbeat` | 把收到的任何内容原样发回（echo）                     |

### 消息格式

每条消息序列化为六个或更多 ZMQ 帧。

```
[b"u-u-i-d",         # zmq identity(ies)
 b"<IDS|MSG>",       # delimiter
 b"baddad42",         # HMAC signature (with key)
 b"{header}",          # header JSON
 b"{parent_header}",   # parent header JSON
 b"{metadata}",        # metadata JSON
 b"{content}",         # content JSON
]
```

详细格式遵循 Jupyter 线路协议标准。每个请求都包裹在 iopub 的 `status: busy` 和 `status: idle` 之间。

### 支持的消息

| 请求                | 回复                                                                                  |
|------------------------|----------------------------------------------------------------------------------------|
| `kernel_info_request`  | `language_info`：name 为 `nabi`，mimetype 为 `text/x-nabi`，file_extension 为 `.nabi`，codemirror_mode 和 pygments_lexer 为 `rust`（在前端认识 Nabi 之前用于高亮）；`help_links` 指向文档（`https://docs.nabi-lang.org/latest/`）和本 REPL 章节；`banner` 为 REPL 的工具链行（[9.7.8](907-jit-cells.md#行编辑)）；`protocol_version` 为 `"5.3"` |
| `execute_request`      | 见下文“执行”                                                                  |
| `is_complete_request`  | 按 REPL 的续行规则判定为 `complete` / `incomplete`；`indent` 为每个仍未闭合的括号四个空格 |
| `complete_request`     | REPL 的补全候选项（9.8.5）                                               |
| `inspect_request`      | 对光标处的名称返回 `found: true` 和 `text/plain` 帮助（9.8.6），否则为 `false`  |
| `history_request`      | 此内核执行过的单元                                                      |
| `comm_info_request`    | 空列表                                                                             |
| `shutdown_request`     | 在 shell 或 control 上。回复后退出                                              |
| `interrupt_request`    | 在 control 上（9.8.4）                                                                     |

### 执行

收到 `execute_request` 后，内核发送：

1. iopub `execute_input`（代码和执行计数）。
2. **随写随发**的单元输出：正在运行的单元写到 stdout / stderr 的内容，在内核读到时作为 iopub `stream`（`stdout` / `stderr`）发送。两者是独立的管道，因此同一管道的各块保持顺序，两个管道的各块按读取顺序到达。在运行之外写出的输出（诊断、`:` 命令）在下一次运行之前或单元结束时发送。
3. 如果有值，发送 iopub `execute_result`：`text/plain` 中是与 REPL 相同的输出（9.7.7）。类型实现了 `RichDisplay`（[标准库 2.10](../../refs-standard-libraries/200-core/210-fmt.md)）的值改为通过 `display` 显示：发送 iopub `display_data`，其 `data` 为该值的 `mime_bundle()`（`image/svg+xml`、`image/png`、`text/html` 等），不发送 `execute_result`。
4. 失败时发送 iopub `error`：`evalue` 是尚未发送的 stderr 的第一行（无法编译的单元的诊断，或关于运行如何结束的报告），`traceback` 是其各行。单元自己写到 stderr 的内容（例如 panic 消息）此前已经流式发送。
5. shell `execute_reply`（包括 `execution_count`，以及成功 / 失败状态）。

输出的 MIME 类型为 `text/plain` 以及 `RichDisplay` 包中的那些类型。`display(x)` 在单元中途发送 `display_data`，位于单元流式输出中它所在的位置：它向 stdout 写入一个带帧标记的包（内核为它运行的单元设置 `NABI_JUPYTER`），内核再把它从文本中取出。不支持原地更新显示（`update_display_data`）。

## 9.8.3. 安装 kernelspec

Jupyter 只识别安装在 kernelspec 目录中的内核。

```
nabi jupyter install [--env <name>]
```

不带参数的 `nabi jupyter` 是用法错误（退出码 `2`），它会说明应当运行什么：`` Jupyter starts the kernel itself; to add it, run `nabi jupyter install` ``。

kernelspec 是下列目录下的 `kernels/nabi/kernel.json`。用 `--env <name>` 安装时，它是 `kernels/nabi-<name>/`，显示名称为 `nabi (<name>)`，命令为 `nabi --env <name> jupyter ...`。

1. 如果设置了 `$JUPYTER_DATA_DIR`，则为它
2. 在 macOS 上为 `~/Library/Jupyter`
3. 在 Windows 上为 `%APPDATA%\jupyter`
4. 其他情况为 `$XDG_DATA_HOME/jupyter`，未设置时为 `~/.local/share/jupyter`

它旁边是 `logo-32x32.png`、`logo-64x64.png` 和 `logo-svg.svg`，即前端为该内核显示的 Nabi 标志。`kernel.json` 运行 `nabi` 代理（[9.10.6](910-nabi-frontend.md#9106-toolchain)），并在 `env` 中记录安装时的 `NABI_HOME` 和 `PATH`。

```json
{
  "argv": ["/home/u/.nabi/bin/nabi", "jupyter", "--connection-file", "{connection_file}"],
  "display_name": "nabi",
  "language": "nabi",
  "interrupt_mode": "message",
  "env": { "NABI_HOME": "/home/u/.nabi", "PATH": "<PATH at install time>" }
}
```

安装后，Nabi 内核可以在 Jupyter、VS Code、nteract 等环境中使用。

## 9.8.4. 中断 / 关闭

### 中断

由于 `interrupt_mode` 为 `"message"`，中断以 control 套接字上的 `interrupt_request` 到达。正在运行的单元被终止，会话从该单元之前的状态继续（与失败的单元相同，[9.7.0](907-jit-cells.md#970-单元会话的规则)）。正在等待 `input_reply` 的单元同样会结束。仍在编译中的单元无法被中断。

### 关闭（内核重启）

在 shell 或 control 套接字上回复 `shutdown_request` 之后，内核清理会话并退出进程。重启时，Jupyter 会自动再次启动它。

## 9.8.5. 代码补全

`complete_request` 把光标前的片段与终端 REPL 的 Tab 补全候选项（[9.7.8](907-jit-cells.md#行编辑)）进行匹配：会话的绑定和项、core 库的名称、会话的 `use` 项导入的名称，以及关键字；`.` 之后为成员，`::` 之后为项。它以 `{ matches, cursor_start, cursor_end }` 回复。

## 9.8.6. inspect_request：帮助

`inspect_request`（Shift+Tab）回复 `found: true`，并以 `text/plain` 给出光标处的名称（形式为 `name: type`）、其签名和文档注释（[词法结构](../001-lexical/102-lexical-structure.md)中的 `///` / `//!`）。对于未知的名称，回复 `found: false`。

## 9.8.7. REPL 与 Jupyter 之间的代码共享

在 REPL 中编写的代码可以原样移到 Jupyter 单元中，反之亦然，因为两者遵循相同的单元会话和相同的语义（[9.7.0](907-jit-cells.md#970-单元会话的规则)）。

### 差异

两种界面在少数方面有所不同，汇总如下。

| 方面      | REPL              | Jupyter                      |
|-------------|-------------------|------------------------------|
| 输入       | 一行 / 一个表达式 | 一个单元（多行） |
| 输出      | 仅文本（`display(x)` 打印其 `text/plain`） | 文本，以及 `RichDisplay` 的各种 MIME 类型 |
| stdin       | 终端；使用管道时为空 | 带 `allow_stdin` 的 `input_request` |
| 状态       | 退出时丢弃 | 单元保存在笔记本文件中 |

## 9.8.8. 安全 / 隔离

内核以用户的权限运行任意代码。不进行任何隔离。

## 9.8.9. 下一步

接下来的[调试 / 诊断工具](909-debug-tools.md)定义便于 gdb 使用的功能、性能分析、`--emit` 等内容。

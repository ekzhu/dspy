# DSPy Python Interpreter 实现文档

## 1. 背景

`dspy.RLM`（Recursive Language Model）等模块需要执行由 LLM 生成的 Python 代码，并允许这些代码调用用户在主进程中定义的 Python 工具函数。这带来两个要求：

- **安全隔离**：LLM 生成的代码默认不能访问文件系统、网络、子进程或环境变量。
- **宿主回调**：沙箱内的 Python 代码需要能调用宿主进程里的 Python 函数（tool），并拿到返回值。

DSPy 的方案是 **Deno + Pyodide**，通过 stdin/stdout 上的 **JSON-RPC 2.0** 实现双向通信。

## 2. 关键技术

| 组件 | 角色 |
|------|------|
| **Deno** | 一个 TypeScript/JavaScript 运行时。默认禁用所有权限，通过 `--allow-read` / `--allow-net` 等白名单显式放行。DSPy 用它做 OS 层的沙箱边界。 |
| **Pyodide** | CPython 编译成 WebAssembly 的版本。提供 JS ↔ Python 的 FFI（`pyodide.runPython`、`pyodide.globals.set`、`pyodide.ffi.run_sync`）。真正的 Python 代码在它内部执行。 |

Deno 作为 JS 运行时天然支持 WebAssembly（所有主流 JS 引擎都带 `WebAssembly` 全局对象），所以它能加载 Pyodide 的 `.wasm` 并执行其中的 Python 解释器。

## 3. 整体架构

```
┌─────────────────────────────┐
│ 宿主 Python 进程              │
│  - 用户工具函数（在内存里）    │
│  - PythonInterpreter 类      │
└───────────┬─────────────────┘
            │ stdin/stdout（JSON-RPC 2.0，每行一条）
┌───────────▼─────────────────┐
│ Deno 子进程 (runner.js)      │
│  - 消息分发 / Pyodide 生命周期 │
└───────────┬─────────────────┘
            │ JS ↔ Python FFI
┌───────────▼─────────────────┐
│ Pyodide (WASM)              │
│  - 执行 LLM 生成的 Python     │
│  - 工具函数以 stub 形式存在   │
└─────────────────────────────┘
```

三层隔离：**用户函数只存在于宿主内存里**，永远不会被序列化或发送到沙箱。

## 4. 核心代码位置

| 文件 | 内容 |
|------|------|
| `dspy/primitives/python_interpreter.py` | 宿主端：启动子进程、发送 JSON-RPC、处理工具回调 |
| `dspy/primitives/runner.js` | Deno 端：管理 Pyodide、分发消息、生成工具 stub |

### 关键函数

- `PythonInterpreter.__init__` (`python_interpreter.py:100-179`)：构造 `deno run --allow-read=...` 命令
- `_register_tools` (`:264-290`)：发送 `register` 请求，带上工具签名元数据
- `_handle_tool_call` (`:292-314`)：收到 `tool_call` 时调用真正的宿主 Python 函数
- `execute` (`:484-563`)：主执行循环，处理嵌套的 tool_call 消息
- `makeToolWrapper` (`runner.js:43-65`)：动态生成 Python 工具 stub
- `toolCallBridge` (`runner.js:151-202`)：沙箱内工具 stub 的底层实现（JS 函数）

## 5. JSON-RPC 协议

每条消息一行 JSON。四种帧：

```jsonc
// 请求（带 id，期待响应）
{"jsonrpc":"2.0","method":"execute","params":{"code":"..."},"id":3}

// 通知（无 id）
{"jsonrpc":"2.0","method":"shutdown"}

// 成功响应
{"jsonrpc":"2.0","result":{"output":"..."},"id":3}

// 错误响应
{"jsonrpc":"2.0","error":{"code":-32007,"message":"..."},"id":3}
```

支持的方法：`execute`、`register`、`mount_file`、`inject_var`、`sync_file`（通知）、`shutdown`（通知）、`tool_call`（由沙箱发起）。

ID 约定：宿主发起的请求用 **整数**（`self._request_id`），沙箱发起的 `tool_call` 用 **字符串** `tc_<timestamp>_<n>`。据此区分方向。

## 6. 工具调用流程（示例）

用户代码：

```python
def lookup(fruit: str) -> str:
    return {"apple": "red"}.get(fruit, "?")

with PythonInterpreter(tools={"lookup": lookup}) as interp:
    interp.execute('print(lookup(fruit="apple"))')
```

内部消息时序：

```
宿主                                                        沙箱
  │── register {lookup(fruit:str)} ──────────────────────▶
  │                                                       runner.js 生成 stub 并 runPython
  │◀── result {tools:["lookup"]} ─────────────────────────
  │── execute "print(lookup(...))" ──────────────────────▶
  │                                                       runPythonAsync → stub → run_sync
  │◀── tool_call {lookup, {fruit:"apple"}} ───────────────
  │   self.tools["lookup"](fruit="apple") → "red"
  │── result {value:"red", type:"string"} ───────────────▶
  │                                                       stub 返回 "red"，Python 继续
  │◀── result {output:"red\n"} ──────────────────────────
  ▼                                                       ▼
```

### 沙箱内的 stub（`makeToolWrapper` 动态生成）

```python
import json
from pyodide.ffi import run_sync, JsProxy
def lookup(fruit: str):
    result = run_sync(_js_tool_call("lookup",
                       json.dumps({"kwargs": {"fruit": fruit}})))
    parsed = result.to_py() if isinstance(result, JsProxy) else result
    if isinstance(parsed, dict) and parsed.get("__dspy_tool_bridge_error__"):
        raise RuntimeError(parsed["message"])
    return parsed
```

- `_js_tool_call` 是 JS 函数，通过 `pyodide.globals.set("_js_tool_call", toolCallBridge)` (`runner.js:205`) 注入到 Python 全局。
- `run_sync` 让同步 Python 代码阻塞等待一个异步 JS Promise —— 用户不用写 `async`。

## 7. 关键设计点

1. **不做序列化**：工具函数的源代码/对象永远不过边界。沙箱只拿到 `{name, parameters}` 元数据，宿主只拿到 JSON 可序列化的参数和返回值。
2. **共享 stdin 读取器**（`runner.js:145`）：`stdinReader` 在模块顶层声明一次，主循环和 `toolCallBridge` 共用，避免两个消费者抢同一条 stdin 导致竞态。
3. **变量注入**：小变量通过 `_serialize_value` 转成 Python 字面量拼到代码前面；超过 100MB 的变量走虚拟文件系统（`/tmp/dspy_vars/<name>.json`），避开 Pyodide FFI 的 128MB 限制。
4. **安全白名单**：Deno 默认禁止文件/网络/子进程；`enable_read_paths`、`enable_network_access` 等参数按需放行。测试里显式验证了沙箱内 `pyfetch` 会被拒。
5. **错误传递**：宿主异常转成 JSON-RPC error（带类型码 `-32000 ~ -32099`），沙箱内 stub 再 `raise RuntimeError`，让 LLM 生成的代码能 `try/except`。

## 8. 与其他方案的对比

| | DSPy (Deno + Pyodide) | Docker 沙箱 (AgentScope 等) | E2B |
|---|---|---|---|
| 宿主函数回调 | **原生支持**（`tools=[fn]`） | 需自建（HTTP/RPC） | 需自建（HTTP/隧道） |
| 沙箱内 Python | Pyodide，受限的 wheel 生态 | 完整 CPython | 完整 CPython |
| 传输 | stdin/stdout pipe | HTTP/gRPC 到本地 Docker | HTTPS/WebSocket 跨网络 |
| 启动成本 | 几百毫秒（Pyodide 冷启动） | 几秒（容器） | 几秒 + 网络 RTT |
| 运维 | 只需要装 Deno | 需要 Docker | 无（但收费） |
| 适用场景 | 本地轻量、快速迭代 | 需要完整 CPython、自托管 | 托管产品、无运维 |

DSPy 的选择偏向 **零基础设施 + 强隔离**，代价是 Pyodide 无法 `pip install` 任意带 C 扩展的包。

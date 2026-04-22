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

## 9. 备选方案：Pydantic Monty

[Pydantic Monty](https://github.com/pydantic/monty) 是 Pydantic 团队 2026 年初开源的一个 **用 Rust 从零实现的 Python 解释器**，专门为运行 LLM 生成的代码而设计。它不是嵌入 CPython，也不是 WASM，而是在 Python 进程内以 Rust 扩展（PyO3）的形式直接跑。

### 为什么吸引人

| 指标 | Pyodide (DSPy 现状) | Monty |
|------|---------------------|-------|
| 冷启动 | 几百毫秒 | **~0.06 ms** |
| 进程数 | Python + Deno 子进程 + WASM | **同进程** |
| 安装体积 | 需装 Deno（~80 MB） | `pip install pydantic-monty`（~4.5 MB） |
| 工具回调 | JSON-RPC + 动态 stub | **原生 `external_functions` 字典** |
| 状态快照 | 无 | **可序列化恢复** |

Monty 的工具调用模型更优雅：沙箱代码调用 `external_functions` 里的函数时，**解释器挂起**并交出一个 `FunctionSnapshot`，宿主执行真正的函数后调用 `resume(value)` 恢复。没有 JSON-RPC、没有管道、没有 stub 代码生成 —— 本质上是 Rust 层的协作式 suspend/resume。

典型 API：

```python
import pydantic_monty

async def call_llm(prompt: str) -> str:
    ...  # 宿主 Python 函数

m = pydantic_monty.Monty(code, inputs=['prompt'])
output = await m.run_async(
    inputs={'prompt': 'testing'},
    external_functions={'call_llm': call_llm},
)
```

### 致命限制（截至 v0.0.3）

Monty 是自研解释器，目前 **只支持 Python 的一个子集**：

- **不支持** `class` 定义、`with` 上下文管理器、`match`、生成器
- **不支持** 任何第三方库（没有 `numpy`、`pandas`、`requests`）
- **标准库极少**：仅 `sys`、`os`、`typing`、`asyncio`、`re`、`datetime`、`json`
- 连内置也不完整：`sorted(key=...)` 不支持

而 `dspy.RLM` 的核心场景恰恰是让 LLM **写 Python 去切片分析长文本** —— LLM 习惯性地写 `with open(...)`、`@dataclass`、`pd.DataFrame`，这些 Monty 目前都跑不了。

另外，全新解释器意味着 **未知的攻击面**（Pydantic 自己开了 $5000 的 bug bounty 来找逃逸漏洞），这也是采用时需要权衡的风险。

### 建议

**不适合做 Pyodide 的直接替代**，但可以作为 `CodeInterpreter` 协议的 **可选后端**（`dspy/primitives/code_interpreter.py` 已经是抽象接口）：

- **适合**：高并发、代码片段短的 agent 服务；无法装 Deno 的 serverless 环境；纯结构化的 tool 编排。
- **不适合**：需要 numpy/pandas 的数据分析；带 class、with、生成器的"真实" Python 代码。

先例：pydantic-ai [PR #4153](https://github.com/pydantic/pydantic-ai/pull/4153) 做过完全相同的抽象，把 Monty、Docker、Local、Memory 并列为可选 `ExecutionEnvironment`，让用户自己挑。DSPy 可以沿用这个思路：默认保留 `PythonInterpreter`（Pyodide），新增 `MontyInterpreter` 作为 opt-in。

## 10. 基于 Docker 的实现思路

如果要让 DSPy 支持一个 **完整 CPython + 完整生态** 的后端，Docker 是最直接的选择。关键是如何保持宿主回调这个核心特性 —— 这在 E2B 这类服务里都是需要自建的。下面是实现这个后端的设计要点。

### 关键挑战：双向通信

Docker 容器和宿主跨进程（通常也跨网络命名空间）。要让容器内代码能调用宿主函数，必须有一条 **容器 → 宿主** 的反向通道。几种可选方案：

| 方案 | 传输 | 优点 | 缺点 |
|------|------|------|------|
| **stdin/stdout**（类似 Deno 方案） | `docker run -i` 挂管道 | 与现有协议同构，改动最小 | 只能一条管道并发受限；容器崩溃难诊断 |
| **Unix domain socket**（挂载到容器） | `-v /host/sock:/sandbox/sock` | 低延迟；明确的 endpoint | 需要额外的守护；权限/所有权问题 |
| **HTTP + 容器网络** | 宿主起 HTTP 服务，容器 `requests.post` | 标准、易调试 | 要管理端口、认证；有网络栈开销 |
| **gRPC** | 同上 | 强类型、双向流 | 依赖重 |

推荐 **stdin/stdout**：可以直接复用现在 `runner.js` 那套 JSON-RPC 协议，语义完全一致，只是把 Deno 子进程换成 Docker 子进程。

### 参考架构

```
┌─────────────────────────────┐
│ 宿主 Python 进程              │
│  - 用户工具函数               │
│  - DockerInterpreter         │
└───────────┬─────────────────┘
            │ stdin/stdout（JSON-RPC 2.0）
            │ docker run -i --rm --network=none ...
┌───────────▼─────────────────┐
│ Docker 容器                  │
│  ├─ runner.py（宿主提供）    │ ← 镜像里预装
│  │   - 读取 stdin JSON-RPC   │
│  │   - 动态生成 tool stub    │
│  │   - exec() 用户代码       │
│  └─ CPython + 预装依赖       │
└─────────────────────────────┘
```

容器镜像里预装一个 `runner.py`，职责和 `runner.js` 对等：

1. 从 stdin 读 JSON-RPC 请求
2. 收到 `register` 时在全局命名空间里 `exec()` 一段生成的 stub（见下）
3. 收到 `execute` 时 `exec(code, globals_dict)`
4. stub 内通过 stdout 发 `tool_call` 请求，然后 **阻塞读 stdin** 等响应

### Tool stub 在容器内的样子

容器内是真正的 CPython，不需要 Pyodide 的 `run_sync` 把戏。直接用同步 I/O：

```python
# 容器内，由 runner.py 通过 exec() 注入
import json, sys, uuid

def _call_host_tool(name, kwargs):
    req_id = f"tc_{uuid.uuid4().hex}"
    sys.stdout.write(json.dumps({
        "jsonrpc": "2.0", "method": "tool_call",
        "params": {"name": name, "kwargs": kwargs},
        "id": req_id,
    }) + "\n")
    sys.stdout.flush()
    # 同步等响应
    line = sys.stdin.readline()
    resp = json.loads(line)
    if resp.get("id") != req_id:
        raise RuntimeError("tool bridge id mismatch")
    if "error" in resp:
        raise RuntimeError(resp["error"]["message"])
    return resp["result"]["value"]

def lookup(fruit: str):
    return _call_host_tool("lookup", {"fruit": fruit})
```

比 Pyodide 那套还简单 —— 没有 WASM FFI、没有 `run_sync`，就是普通 Python 读写两条管道。

### 安全隔离

Docker 层的沙箱配置是重点，推荐默认值：

```bash
docker run -i --rm \
  --network=none \                       # 默认禁网
  --read-only \                          # 只读根文件系统
  --tmpfs /tmp:size=100m \               # 只给少量可写空间
  --memory=512m --cpus=1.0 \             # 资源上限
  --pids-limit=128 \                     # 防 fork 炸弹
  --security-opt=no-new-privileges \     # 禁止 setuid 提权
  --cap-drop=ALL \                       # 丢掉所有 Linux capabilities
  dspy-sandbox:latest
```

进阶加固：

- **gVisor / Kata Containers**：把内核攻击面也隔掉，代价是启动更慢
- **seccomp profile**：只允许 Python 运行必需的 syscall 白名单
- **rootless Docker**：host 侧 uid 隔离

白名单机制和当前 `PythonInterpreter` 的 `enable_network_access` / `enable_read_paths` 对齐：默认禁用，按需 `--network=host` 或 `-v` 挂载放行。

### 状态和启动成本

Docker 方案有两种模式：

1. **一次性容器**（每次 `execute` 新启一个）：最干净，但冷启动 1–3 秒
2. **长连接容器**（容器里跑一个 REPL，多次 `execute` 共用）：类似 Jupyter 内核，状态跨调用保留，冷启动只付一次

对 `dspy.RLM` 这种迭代式的执行循环，**长连接模式** 更合适。需要额外处理：

- 容器健康检查（进程死了要自动重启）
- 超时控制（单次 `execute` 跑太久要能 kill）
- 并发隔离（每个 `RLM` 实例一个容器，或池化）

### 实现清单

如果真要在 DSPy 里加一个 `DockerInterpreter`，大致工作量：

1. 实现 `dspy/primitives/docker_interpreter.py`，实现 `CodeInterpreter` 协议（`__init__` / `execute` / `shutdown`）
2. 写 `dspy/primitives/docker_runner.py`，打进 `dspy-sandbox:<version>` 镜像，推到 Docker Hub 或让用户自己 build
3. 协议完全复用 `python_interpreter.py` 里的 `_jsonrpc_*` helper —— 消息格式保持一致，这样 `CodeInterpreter` 协议层的调用方不用区分后端
4. 加参数：`image`、`memory_limit`、`cpu_limit`、`network`、`reuse_container`、`docker_command`（类比 `deno_command`）
5. 测试：与现有 Pyodide 测试共享一批用例（工具调用、变量注入、错误传播），外加 Docker 特有的资源限制测试

### 何时选 Docker 方案

- 用户需要 `numpy`、`pandas`、`scikit-learn` 甚至自定义 `pip install`
- 需要严格的 CPU / 内存 / PID 配额（生产环境多租户）
- LLM 生成的代码会用到 Pyodide 跑不动的模式（C 扩展、子进程、完整 stdlib）

**不值得** 的场景：本地开发、单用户 notebook、代码片段简短 —— 这些情况 Pyodide 更轻量，或者 Monty 更快。

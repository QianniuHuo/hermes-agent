# Hermes Agent 全局架构扫描文档

> 版本: 1.0 | 源码分支: 当前最新
> 角色: 资深软件架构师 & 源码逆向专家

---

## 目录

1. [执行管线 (Execution Pipeline)](#1-执行管线-execution-pipeline)
2. [委托机制 (Delegation)](#2-委托机制-delegation)
3. [记忆分层系统 (Memory Layers)](#3-记忆分层系统-memory-layers)
4. [技能与工具演化 (Skills & Tools Evolution)](#4-技能与工具演化-skills--tools-evolution)
5. [环境隔离 (Environment Isolation)](#5-环境隔离-environment-isolation)
6. [核心组件交互流程图](#6-核心组件交互流程图)
7. [核心模块职能说明](#7-核心模块职能说明)

---

## 1. 执行管线 (Execution Pipeline)

### 1.1 入口点

| 文件 | 入口函数/类 | 职责 |
|------|------------|------|
| `hermes_cli/main.py:1` | `main()` | CLI 入口，`_apply_profile_override()` 设置 HERMES_HOME |
| `cli.py:1590` | `HermesCLI` | 交互式 TUI/REPL，`run()` 启动循环 |
| `run_agent.py:535` | `AIAgent` | 核心 Agent 逻辑 |
| `gateway/run.py` | `HermesGateway` | 消息网关（多平台适配器） |

**启动链路：**
```
hermes chat
  → hermes_cli/main.py::main()
  → _apply_profile_override() 设置 HERMES_HOME
  → 加载配置，路由子命令
  → HermesCLI.run() → 实例化 AIAgent
```

### 1.2 主循环 (Main Loop)

**精确位置：** `run_agent.py:8646`

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) \
      or self._budget_grace_call:
```

**迭代变量：**
- `api_call_count` — 每次 API 调用前递增 (line 8658)
- `self.max_iterations` — 默认 90 (line 563)
- `self.iteration_budget` — `IterationBudget` 实例 (line 659)，线程安全
- `_budget_grace_call` — 允许预算耗尽后额外一次调用 (line 822-823)

**循环结构 (lines 8646–9536+):**

```
1. 中断检查 (line 8651) — 用户请求中断则 break
2. 消耗预算 (line 8667) — iteration_budget.consume()
3. 构建 API 消息 (line 8706–8822)
4. 调用 LLM — _interruptible_streaming_api_call() (line 8993)
   或 _interruptible_api_call() (line 8997)
5. 响应验证 — 错误处理、限速、重截断
6. 工具调用提取 — response.tool_calls
7. 工具执行 — _execute_tool_calls_concurrent() (line 7485)
   或 _execute_tool_calls_sequential() (line 7752)
8. 结果追加 — tool_msg 追加到 messages[] (line 7739–7744)
9. 循环 — 直至 finish_reason == "stop" 或预算耗尽
```

### 1.3 工具编排 (Tool Orchestration)

**`model_tools.py` 关键函数：**

| 函数 | 位置 | 职责 |
|------|------|------|
| `discover_builtin_tools()` | line 132 | 扫描 `tools/` 目录，导入所有含 `registry.register()` 的模块 |
| `get_tool_definitions()` | line 196 | 返回过滤后的工具 schema 列表 |
| `handle_function_call()` | line 421 | 主分发器 — 路由至 registry |
| `coerce_tool_args()` | line 334 | 类型强制转换 (e.g. "42" → 42) |

**`tools/registry.py` 关键方法：**

| 方法 | 行号 | 职责 |
|------|------|------|
| `register()` | 176 | 注册工具（schema + handler + check_fn） |
| `deregister()` | 229 | 注销工具 |
| `get_definitions()` | 258 | 返回 OpenAI-format schemas |
| `dispatch()` | 292 | 执行工具 handler |
| `discover_builtin_tools()` | 56–73 | AST 扫描 `tools/*.py` |

### 1.4 执行流程图

```
用户输入
    │
    ▼
HermesCLI.run()
    │
    ▼
AIAgent.run_conversation()  ──► line 8284
    │
    ├─► 构建系统提示词 (8451–8467)
    ├─► 压缩前检查 (8499–8558)
    └─► 用户消息追加至 messages[] (8432)
            │
            ▼
    ┌─────── MAIN LOOP (8646) ◄───────────────────┐
    │                                              │
    │  api_call_count++ (8658)                     │
    │  iteration_budget.consume() (8667)          │
    │  构建 API 消息 (8706–8822)                    │
    │  _interruptible_streaming_api_call() (8993)  │
    │       │                                      │
    │       ▼                                      │
    │   LLM 响应                                    │
    │       │                                      │
    │       ▼                                      │
    │   finish_reason = response.choices[0].       │
    │       finish_reason                         │
    │                                              │
    ├─► 有 tool_calls:                             │
    │       │                                      │
    │       ├─► _execute_tool_calls_concurrent()  │
    │       │     (7485) 或                        │
    │       │     _execute_tool_calls_sequential()│
    │       │     (7752)                          │
    │       │       │                              │
    │       │       ├─► _invoke_tool() (7373)    │
    │       │       │       │                      │
    │       │       │       ▼                      │
    │       │       │   handle_function_call()     │
    │       │       │       │                      │
    │       │       │       ▼                      │
    │       │       │   registry.dispatch()        │
    │       │       │       │                      │
    │       │       │       ▼                      │
    │       │       │   工具 handler 返回 JSON      │
    │       │       │       │                      │
    │       │       ▼       │
    │       └─► tool_msg 追加至 messages[] ────────┘
    │           (7739–7744)
    │
    └─► 无 tool_calls / finish_reason == "stop":
            │
            ▼
        返回 final_response
```

---

## 2. 委托机制 (Delegation)

### 2.1 概述

**模式：层级星型 (Hierarchical Star) — 中心化父节点调度，非 Swarm 对等模式**

- 父 Agent 是唯一入口点 — 决定何时、委托什么
- 子 Agent 相互隔离 — 不允许跨代通信（`MAX_DEPTH = 2`，line 53）
- 子 Agent 之间无直接通信 — 并行模式下独立执行，结果汇总
- 无节点发现机制 — 无注册中心，无网状组网
- 结果以**摘要字符串**形式返回父上下文（而非原始对话历史）

### 2.2 派生逻辑 (`tools/delegate_tool.py`)

**入口函数：** `delegate_task()` at **line 623**

**子 Agent 构建：** `_build_child_agent()` at **lines 238–397**

**关键步骤：**

| 步骤 | 行号 | 行为 |
|------|------|------|
| 1 | 714–726 | `delegate_task()` 调用 `_build_child_agent()` |
| 2 | 348–377 | 创建 AIAgent：受限 system_prompt、restricted toolsets、`quiet_mode=True`、`skip_context_files=True`、`skip_memory=True` |
| 3 | 380 | `_delegate_depth = parent_depth + 1`（上限 `MAX_DEPTH=2`） |
| 4 | 472 | `child.run_conversation(user_message=goal)` — **阻塞调用** |
| 5 | 741 | 批量任务使用 `ThreadPoolExecutor` 并行执行 |

### 2.3 与父 Agent 的特殊耦合

`delegate_task` **不在** `model_tools.py` 中特殊处理，而是**在 `run_agent.py` 中绕过** `handle_function_call()` 直接调用：

- **line 326** — `delegate_task` 列入 `_AGENT_LOOP_TOOLS`，若到达通用分发器则返回错误 stub
- **lines 7441–7450, 7927–7959** — Agent 主循环**直接 import 并调用** `delegate_task`

原因：`delegate_task` 需要访问 `parent_agent`（完整 AIAgent 实例）以：
- 读取父的 `enabled_toolsets`、`valid_tool_names`
- 继承凭证、提供商配置
- 在 `_active_children` 注册子进程用于中断传播
- 共享凭证池

### 2.4 `_last_resolved_tool_names` 保护协议

此全局变量（`model_tools.py:157–159`）存储最近一次 `get_tool_definitions()` 的工具名列表。`execute_code` 工具依赖此变量确定沙箱可导入哪些工具。

**委托时的保存/恢复协议：**

| 步骤 | 位置 | 行为 |
|------|------|------|
| 子构建前 | `delegate_task()` line 706 | `_parent_tool_names = list(_model_tools._last_resolved_tool_names)` |
| 子构建中 | `_build_child_agent()` line 725 | `child._delegate_saved_tool_names = _parent_tool_names` |
| 子构建后 | `delegate_task()` line 729 | `_model_tools._last_resolved_tool_names = _parent_tool_names`（权威恢复） |
| 子运行中 | `_run_single_child()` line 418–419 | 通过 `child._delegate_saved_tool_names` 读取 |
| 子运行后 | `_run_single_child()` line 596–598 | `model_tools._last_resolved_tool_names = list(saved_tool_names)` |

### 2.5 父子通信机制

**父 → 子：**
- `goal` 字符串 → 子的 `user_message`
- `context` 字符串 → 嵌入子的 `ephemeral_system_prompt`
- `toolsets` → 子的 `enabled_toolsets`（受限/交集）
- `parent_agent` 对象引用 → 凭证继承、会话共享、进度回调

**子 → 父：**
- `child.run_conversation()` 返回 `{"final_response": summary_string}`
- 工具追踪从子的内存消息构建（lines 500–534），不对父暴露
- 仅 `summary`（字符串）进入父上下文

### 2.6 上下文隔离

**每个子 Agent 拥有：**
- 独立对话 — `run_conversation()` 从零开始，无历史
- 独立 `task_id` — 独立终端会话、文件操作缓存
- **无 memory** — `skip_memory=True`（line 366）
- **无 context files** — `skip_context_files=True`（line 365）
- 受限 toolsets — 与父 enabled_tools 取交集（lines 270–291）
- `DELEGATE_BLOCKED_TOOLS`（line 32–38）必剔除：`delegate_task`, `clarify`, `memory`, `send_message`, `execute_code`
- 独立迭代预算 — `iteration_budget=None`（line 376）

**共享资源：**
- `session_db`（SQLite）— 传入引用用于会话持久化
- `parent_session_id` — 关联父子会话
- `_credential_pool` — 共享凭证池用于限速轮换
- API 凭证 — 继承自父，除非显式覆盖

### 2.7 进程级隔离

**无硬进程隔离** — 子 Agent 在同一 Python 进程内作为线程运行（`ThreadPoolExecutor`，line 741）。

**共享状态风险缓解：**
- `_active_children_lock` + `_active_children` 列表用于中断传播
- `_last_resolved_tool_names` 保存/恢复协议防止子污染父
- `child.close()` 在 `finally` 块（line 617–621）清理终端沙箱、浏览器守护进程、后台进程
- `MAX_DEPTH = 2` 防止无界递归
- 心跳线程（lines 437–469）保持父网关超时活跃

---

## 3. 记忆分层系统 (Memory Layers)

### 3.1 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| `SessionDB` | `hermes_state.py:115` | SQLite 持久化会话存储 + FTS5 全文搜索 |
| `MemoryStore` | `tools/memory_tool.py:105` | MEMORY.md/USER.md 文件记忆（带快照模式） |
| `TodoStore` | `tools/todo_tool.py:25` | 会话内 Todo 列表（内存，非持久化） |
| `MemoryProvider` | `agent/memory_provider.py:42` | 可插拔记忆提供者抽象基类 |
| `MemoryManager` | `agent/memory_manager.py:83` | 协调内置 + 一个外部提供者 |
| `ContextCompressor` | `agent/context_compressor.py` | 长对话压缩/摘要 |

### 3.2 会话持久化流程

#### CLI 模式 (`run_agent.py`)

```
会话启动:
  run_agent.__init__()
    → 创建 _session_db (SessionDB)
    → create_session() at line ~1100

对话中:
  _persist_session() (line 2500)
    → _save_session_log() — 写 JSONL
    → _flush_messages_to_session_db() — 写 SQLite (lines 2507–2553)
      — 使用 _last_flushed_db_idx 去重 (bug #860 修复)

会话结束 (/new, /reset, exit):
  → commit_memory_session() — 调用 memory_manager.on_session_end()
  → _session_db.end_session() — 标记会话结束
  → 若压缩：会话轮换，新 ID，parent_session_id 关联
```

#### Gateway 模式 (`gateway/run.py` + `gateway/session.py`)

```
消息到达:
  _process_message()
    → session_store.get_or_create_session()
    → session_store.load_transcript() — 从 SQLite（优先）或 JSONL（兼容）加载
    → 使用较多消息源（防止静默截断）

Agent 响应后:
  → Agent 已调用 _flush_messages_to_session_db()（gateway 跳过去重写入）
  → session_store.append_to_transcript() 写 JSONL（向后兼容）
  → session_store.update_session() — 更新时间戳

会话重置（空闲/每日策略）:
  → get_or_create_session() 检测陈旧会话
  → 结束旧会话，创建新 SessionEntry
```

### 3.3 短时记忆 vs 长时记忆

| 类型 | 范围 | 持久化方式 |
|------|------|-----------|
| `messages` list (run_agent) | 当前会话 | 从 SQLite/JSONL 恢复 |
| `TodoStore` | 当前会话 | 从对话历史中 last todo tool response 重建 |
| `_session_db` messages | 所有会话 | 持久化到 SQLite + FTS5 |
| MEMORY.md / USER.md | Profile 范围（HERMES_HOME） | 文件持久化 |
| 外部记忆提供者（Honcho, Mem0） | 提供商特定 | 插件控制 |
| `session_search` tool | 所有会话 | SQLite FTS5 + LLM 摘要 |

### 3.4 记忆写入 → 存储 → 检索流程

```
写入记忆:

用户消息 / Assistant 响应
    │
    ▼
API 调用完成
    │
    ▼
CLI: _persist_session() 或 Gateway: session_store.append_to_transcript()
    │
    ▼
_flush_messages_to_session_db()
    │
    ▼
SessionDB.append_message() → SQLite + FTS5 索引更新
    │
    ▼
MemoryManager.sync_all() → 通知外部提供者
```

```
读取记忆（系统提示构建）:

会话启动时 → _build_system_prompt()
  ├─ SOUL.md (身份)
  ├─ MEMORY.md 冻结快照 (_memory_store.format_for_system_prompt("memory"))
  ├─ USER.md 冻结快照 (_memory_store.format_for_system_prompt("user"))
  ├─ 外部记忆提供者块 (_memory_manager.build_system_prompt())
  └─ 其他上下文...

每轮前（prefetch）:

MemoryManager.prefetch_all(user_message)
    │
    ▼
每个 provider 返回上下文
    │
    ▼
build_memory_context_block() 包装在 <memory-context> 围栏中
    │
    ▼
注入当前 user message（仅 API 调用时，不持久化，不进入对话历史）
```

### 3.5 优先级排序

**系统提示构建顺序 (`run_agent.py:3396–3561`)：**

1. SOUL.md（身份）或 DEFAULT_AGENT_IDENTITY
2. 工具指导（memory, session_search, skills）
3. Nous 订阅提示
4. 工具使用强制指导
5. 用户/网关系统提示
6. **MEMORY.md block**（若 `memory_enabled` 且 `memory` 在 `valid_tool_names`）
7. **USER.md block**（若 `user_profile_enabled`）
8. **外部记忆提供者 block**
9. Skills 提示
10. Context files
11. 时间戳 + 模型/提供商信息
12. 环境提示
13. 平台特定提示

**轮次召回注入 (`run_agent.py:8720–8731`)：**
- 仅注入当前 user message
- 顺序：外部 provider prefetch context → 插件 user_context
- 包装在 `<memory-context>` 围栏中

### 3.6 Session vs Memory 区分

| 维度 | Session | Memory |
|------|---------|--------|
| 内容 | 对话历史（所有消息） | 提炼的事实/笔记 |
| 粒度 | 原始 user/assistant/tool 消息 | 显式保存的信息 |
| 接口 | 自动（每次 API 调用） | 显式（`memory` 工具调用） |
| 搜索 | `session_search` tool + FTS5 | `memory` tool |
| 格式 | OpenAI message format | MD 文件中的分隔条目 |
| 限制 | 无（由上下文窗口/压缩界定） | 字符限制（memory 2200, user 1375） |
| 压缩 | 通过 context_compressor 摘要 | 冻结快照不压缩 |
| 关键区分 | 存储**对话本身**（所有说过的话） | 存储**知识**（值得记住的事实） |

---

## 4. 技能与工具演化 (Skills & Tools Evolution)

### 4.1 工具注册模式 (`tools/registry.py`)

**每个工具文件（如 `tools/file_tools.py`）通过以下方式自注册：**

```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("SOME_API_KEY"))

def my_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={
        "name": "my_tool",
        "description": "...",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "..."}
            },
            "required": ["param"]
        }
    },
    handler=lambda args, **kw: my_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["SOME_API_KEY"],
    emoji="🔧",
)
```

**`ToolEntry` 结构 (`tools/registry.py:76–97`)：**
```python
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

### 4.2 工具发现机制

**`discover_builtin_tools()` (`tools/registry.py:56–73`)：**
- 使用 AST 分析扫描 `tools/*.py`
- 仅导入含顶层 `registry.register()` 调用的模块
- 排除 `__init__.py`, `registry.py`, `mcp_tool.py`
- 无需手动维护导入列表

**`model_tools.py:132` 触发发现：**
```python
discover_builtin_tools()  # 导入所有 tools/*.py 模块
```

### 4.3 核心工具列表 (`toolsets.py:31–63`)

```python
_HERMES_CORE_TOOLS = [
    # Web
    "web_search", "web_extract",
    # Terminal + process management
    "terminal", "process",
    # File manipulation
    "read_file", "write_file", "patch", "search_files",
    # Vision + image generation
    "vision_analyze", "image_generate",
    # Skills
    "skills_list", "skill_view", "skill_manage",
    # Browser automation
    "browser_navigate", "browser_snapshot", "browser_click",
    "browser_type", "browser_scroll", "browser_back",
    "browser_press", "browser_get_images",
    "browser_vision", "browser_console",
    # Text-to-speech
    "text_to_speech",
    # Planning & memory
    "todo", "memory",
    # Session history search
    "session_search",
    # Clarifying questions
    "clarify",
    # Code execution + delegation
    "execute_code", "delegate_task",
    # Cronjob management
    "cronjob",
    # Cross-platform messaging
    "send_message",
    # Home Assistant
    "ha_list_entities", "ha_get_state", "ha_list_services", "ha_call_service",
]
```

### 4.4 Toolset 定义 (`toolsets.py:66–397`)

Toolsets 是字典结构：
```python
{
    "description": "...",
    "tools": ["tool1", "tool2"],        # 直接列表
    "includes": ["other_toolset"],     # 组合引用
}
```

**关键函数：**
- `resolve_toolset(name)` (line 447) — 递归解析，支持菱形依赖和循环检测
- `get_toolset(name)` (line 401) — 获取定义，同时查询插件 toolsets
- `resolve_multiple_toolsets(toolset_names)` (line 500) — 合并多个 toolsets

### 4.5 添加新工具

**仅需修改 2 个文件：**

**1. 创建 `tools/your_tool.py`**

**2. 添加到 `toolsets.py`：**
- 核心工具 → 添加到 `_HERMES_CORE_TOOLS` (line 31)
- 平台特定 → 添加到对应 platform toolset（如 `"hermes-cli"`, `"hermes-telegram"`）
- 或在 `TOOLSETS` 字典 (line 68) 中定义新 toolset

**自动发现** — 无需手动导入列表。

### 4.6 动态优化（Evolve）能力

"动态优化"在此代码库中体现为：

1. **工具可用性检查** — `check_fn` 在每次 `get_definitions()` 调用时过滤不可用工具
2. **环境变量按需加载** — `requires_env` 指定必需环境变量
3. **Schema 驱动** — 所有工具由 schema 描述，模型可感知能力边界
4. **Toolset 动态解析** — `resolve_toolset()` 支持运行时组合

---

## 5. 环境隔离 (Environment Isolation)

### 5.1 目录结构

```
tools/environments/
├── __init__.py        # 导出 BaseEnvironment
├── base.py            # 抽象基类 + 共享工具
├── local.py           # 本地主机执行
├── docker.py          # Docker 容器执行
├── ssh.py             # SSH 远程执行
├── singularity.py     # Singularity/Apptainer 容器
├── modal.py           # Modal 云沙箱
├── daytona.py         # Daytona 云沙箱
├── modal_utils.py     # Modal 辅助函数
├── managed_modal.py   # Managed Modal 网关
└── file_sync.py       # 远程后端文件同步
```

### 5.2 BaseEnvironment 抽象 (`base.py:252–598`)

**关键抽象：**
- `_run_bash()` — 抽象方法，生成 bash 进程
- `execute()` — 统一命令执行，含 snapshot sourcing
- `init_session()` — 捕获 shell 环境快照
- `_wrap_command()` — 命令包装（含 snapshot sourcing + CWD 标记）
- `_wait_for_process()` — 轮询等待，含中断检查 + 活动心跳
- `cleanup()` — 资源释放抽象方法

**ProcessHandle 协议 (`base.py:151–166`)：**
```python
class ProcessHandle(Protocol):
    def poll(self) -> int | None: ...
    def kill(self) -> None: ...
    def wait(self, timeout: float | None = None) -> int: ...
    @property
    def stdout(self) -> IO[str] | None: ...
    @property
    def returncode(self) -> int | None: ...
```

### 5.3 各后端实现对比

#### LocalEnvironment (`local.py:216–314`)

| 特性 | 实现 |
|------|------|
| 执行 | 直接 `subprocess.Popen` 在宿主机 |
| 环境 | `_sanitize_subprocess_env()` 清理 70+ 提供商环境变量 |
| HOME 隔离 | 每个 profile `get_subprocess_home()` 重定向 |
| 进程终止 | `os.killpg()` 杀进程组 |
| CWD | 命令后通过文件读取 |
| Shell | Windows 用 Git Bash，Unix 用系统 bash |

#### DockerEnvironment (`docker.py:235–578`)

| 特性 | 实现 |
|------|------|
| 安全 | `cap-drop ALL`, `no-new-privileges`, PID 限制 (256), tmpfs (`/tmp`, `/var/tmp`, `/run`) |
| Capabilities | 添加 `DAC_OVERRIDE`, `CHOWN`, `FOWNER` 用于绑定挂载 |
| Workspace | 绑定挂载或 tmpfs（可配置） |
| 持久化 | `~/.hermes/sandboxes/docker/{task_id}/` 下的绑定挂载沙箱目录 |
| 凭证 | 凭证文件、技能目录、缓存目录只读挂载 |
| 网络 | 可配置（默认开启，可关闭） |
| 容器生命周期 | `sleep infinity` + 显式清理 |

#### SSHEnvironment (`ssh.py:31–275`)

| 特性 | 实现 |
|------|------|
| 连接 | SSH ControlMaster 复用 (`ControlPersist=300`) |
| 文件同步 | `FileSyncManager` 通过 tar-over-SSH 批量上传 |
| CWD | 带内 stdout 标记解析 |
| 远程 HOME | 自动检测 `echo $HOME` |

#### 容器化后端 (Modal, Daytona, Singularity)

均使用 `_ThreadedProcessHandle` 适配器 (`base.py:169–235`)：
- 将阻塞 SDK 调用包装在线程中
- 暴露 `ProcessHandle` 兼容接口
- `cancel_fn` 用于后端特定取消

### 5.4 环境选择

**`terminal_tool.py:602`：**
```python
env_type = os.getenv("TERMINAL_ENV", "local")
```
支持值：`local`, `docker`, `singularity`, `modal`, `daytona`, `ssh`

### 5.5 危险命令检测 (`tools/approval.py`)

**危险模式类别 (`approval.py:76–139`)：**

| 类别 | 示例 |
|------|------|
| 破坏性文件操作 | `rm -rf /`, `rm -r`, 递归 `chmod 777` |
| 权限提升 | `chown -R root`, fork bombs |
| 系统修改 | `mkfs`, `dd`, 写 `/dev/sd*`, `/etc/` |
| SQL 破坏性 | 无 WHERE 的 `DROP TABLE`, `DELETE FROM` |
| 服务控制 | `systemctl stop/restart`, `kill -9 -1`, `pkill -9` |
| Shell 注入 | `bash -c`, `python -e`, `curl\|sh`, heredoc exec |
| 网关保护 | `hermes gateway stop/restart`, 自终止 |
| Git 破坏性 | `git reset --hard`, `git push --force`, `git clean -f` |
| 脚本执行 | `chmod +x` 后立即执行 |

**审批流程 (`check_all_command_guards`, lines 694–953)：**

```
1. 容器跳过 — Docker/Singularity/Modal/Daytona 跳过所有检查 (line 704)
2. YOLO 模式 — HERMES_YOLO_MODE 或 /yolo 网关命令绕过
3. Phase 1 — 检测:
   ├─ Tirith 内容安全扫描（若安装）
   └─ 模式匹配 via detect_dangerous_command()
4. Phase 2 — 智能审批 (approvals.mode=smart):
   └─ 辅助 LLM 评估风险 → approve / deny / escalate
5. Phase 3 — 用户审批:
   ├─ CLI: 交互提示 [o]nce / [s]ession / [a]lways / [d]eny
   └─ Gateway: 队列阻塞 + /approve + /approve all
```

**Tirith 安全扫描 (`tirith_security.py`)：**
- 二进制文件自动下载（GitHub）+ SHA-256 校验
- Cosign 来源验证（若可用）
- 内容级威胁：同形异义词 URL、管道解释器、终端注入

---

## 6. 核心组件交互流程图

### 6.1 整体架构 Mermaid 图

```mermaid
flowchart TB
    subgraph CLI["CLI 入口 (hermes_cli/main.py)"]
        main[main()] --> profile[_apply_profile_override()]
        profile --> cliload[load_cli_config()]
        cliload --> hermes_cli[HermesCLI]
    end

    subgraph Gateway["消息网关 (gateway/run.py)"]
        gw_main[HermesGateway]
        subgraph Platforms["平台适配器"]
            tg[Telegram]
            discord[Discord]
            slack[Slack]
            whatsapp[WhatsApp]
        end
        gw_main --> Platforms
    end

    subgraph AgentCore["Agent 核心 (run_agent.py)"]
        ai[AIAgent]
        main_loop[主循环<br/>line 8646]
        iter_budget[IterationBudget<br/>line 659]
        msg_list[messages[]]
        prefetch[MemoryManager.prefetch_all]
        compressor[ContextCompressor]
    end

    subgraph ToolLayer["工具层 (model_tools.py)"]
        tool_defs[get_tool_definitions<br/>line 196]
        handle_fc[handle_function_call<br/>line 421]
        discover[discover_builtin_tools<br/>line 132]
    end

    subgraph Registry["工具注册表 (tools/registry.py)"]
        reg[ToolRegistry<br/>singleton]
        reg_dispatch[registry.dispatch<br/>line 292]
        tool_entries[ToolEntry...]
    end

    subgraph Tools["工具实现 (tools/*.py)"]
        terminal[terminal_tool.py]
        delegate[delegate_tool.py]
        memory[memory_tool.py]
        todo[todo_tool.py]
        file_ops[file_tools.py]
        browser[browser_tool.py]
        web[web_tools.py]
        code_ex[code_execution_tool.py]
    end

    subgraph EnvBackend["执行后端 (tools/environments/)"]
        local[LocalEnvironment]
        docker[DockerEnvironment]
        ssh[SSHEnvironment]
        modal[ModalEnvironment]
        daytona[DaytonaEnvironment]
    end

    subgraph Persistence["持久化层"]
        session_db[(SessionDB<br/>SQLite + FTS5<br/>hermes_state.py)]
        memory_files[(MEMORY.md<br/>USER.md)]
        jsonl_log[session.jsonl]
    end

    subgraph MemorySys["记忆系统"]
        mem_provider[MemoryProvider<br/>agent/memory_provider.py]
        mem_manager[MemoryManager<br/>agent/memory_manager.py]
        todo_store[TodoStore]
    end

    subgraph Delegation["委托系统 (tools/delegate_tool.py)"]
        delegate_task[delegate_task<br/>line 623]
        build_child[_build_child_agent<br/>lines 238-397]
        run_child[_run_single_child<br/>lines 399-621]
        active_kids[_active_children]
        saved_tool_names[_last_resolved_tool_names<br/>保护协议]
    end

    subgraph Approval["安全审批 (tools/approval.py)"]
        approval[check_all_command_guards<br/>line 694]
        tirith[Tirith scanner]
        patterns[DANGEROUS_PATTERNS]
    end

    CLI -->|AIAgent 实例| ai
    Gateway -->|AIAgent 实例| ai

    ai --> main_loop
    main_loop --> iter_budget
    main_loop -->|API 调用| tool_defs
    main_loop -->|每轮前| prefetch
    prefetch --> mem_manager

    tool_defs --> discover
    discover -->|AST 扫描| Tools
    handle_fc --> reg_dispatch
    reg_dispatch -->|查找| tool_entries

    reg --> Tools
    terminal --> EnvBackend
    delegate --> Delegation
    memory --> memory_files
    todo --> todo_store

    EnvBackend --> Approval
    Approval -->|危险命令| patterns
    Approval -->|Tirith| tirith

    local --> docker
    local --> ssh

    ai --> session_db
    ai --> jsonl_log
    mem_manager --> session_db
    mem_manager --> memory_files
    mem_manager --> mem_provider

    delegate_task --> build_child
    build_child --> run_child
    build_child --> saved_tool_names
    run_child --> active_kids
```

### 6.2 主循环详细 Mermaid 图

```mermaid
sequenceDiagram
    participant User as 用户输入
    participant CLI as HermesCLI
    participant Agent as AIAgent
    participant Loop as 主循环 (8646)
    participant LLM as API Provider
    participant MT as model_tools
    participant Reg as ToolRegistry
    participant Tool as 工具 handler
    participant Mem as MemoryManager
    participant DB as SessionDB

    User->>CLI: chat(message)
    CLI->>Agent: run_conversation(message)

    Note over Agent: 构建系统提示词<br/>8451-8467

    Agent->>Mem: prefetch_all(user_message)
    Mem-->>Agent: memory_context_block

    Agent->>Loop: while 条件满足
    Loop->>Agent: iteration_budget.consume()

    Note over Loop: 构建 API 消息<br/>8706-8822

    Loop->>LLM: chat.completions.create()
    LLM-->>Loop: response

    alt 有 tool_calls
        Loop->>Loop: _execute_tool_calls_concurrent()
        loop 每个 tool_call
            Loop->>MT: handle_function_call(name, args)
            MT->>Reg: dispatch(name, args)
            Reg->>Tool: handler(args)
            Tool-->>Reg: JSON result
            Reg-->>MT: result
            MT-->>Loop: result
            Loop->>Agent: messages.append(tool_msg)
        end
    else finish_reason == "stop"
        Loop-->>Agent: return response.content
    end

    Agent->>DB: _flush_messages_to_session_db()
    DB-->>Agent: persisted

    Agent-->>CLI: final_response
    CLI-->>User: 输出
```

### 6.3 委托流程 Mermaid 图

```mermaid
flowchart LR
    subgraph Parent["父 AIAgent"]
        p_loop[主循环]
        p_dispatch[工具分发]
    end

    subgraph Delegate["delegate_tool.py"]
        dt[delegate_task<br/>line 623]
        bc[_build_child_agent<br/>lines 238-397]
        rc[_run_single_child<br/>lines 399-621]
        tool_names_prot[_last_resolved_tool_names<br/>保护协议]
    end

    subgraph Child["子 AIAgent"]
        c_loop[独立主循环]
        c_tools[受限 toolsets]
    end

    p_dispatch--|"delegate_task<br/>绕过 handle_function_call"| dt
    dt-->|"构建 child_agent"| bc
    bc-->|"MAX_DEPTH 检查<br/>工具集交集"| c_tools
    bc-->|"skip_memory=True<br/>skip_context_files=True"| c_loop

    rc-->|"child.run_<br/>conversation()"| c_loop
    c_loop-->|"final_response<br/>summary"| rc

    rc-->|"close() 清理"| c_loop
    dt-->|"_active_children<br/>中断传播"| rc

    bc -.->|"_delegate_saved<br/>_tool_names"| tool_names_prot
    rc -.->|"restore<br/>_last_resolved<br/>_tool_names"| tool_names_prot
```

---

## 7. 核心模块职能说明

### 7.1 入口与配置

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `hermes_cli/main.py` | `main()`, `_apply_profile_override()` | CLI 入口，HERMES_HOME profile 隔离 |
| `hermes_cli/config.py` | `DEFAULT_CONFIG`, `load_config()` | 配置定义与加载，含迁移逻辑 |
| `hermes_cli/skin_engine.py` | `SkinConfig`, `init_skin_from_config()` | CLI 皮肤/主题引擎 |
| `cli.py` | `HermesCLI` (line 1590) | 交互式 TUI，slash 命令分发，Rich + prompt_toolkit |

### 7.2 Agent 核心

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `run_agent.py:535` | `AIAgent` | 主 Agent 类，`run_conversation()` 核心循环 |
| `run_agent.py:8646` | 主循环条件 | `while (api_call_count < max_iterations and iteration_budget.remaining > 0)` |
| `run_agent.py:659` | `IterationBudget` | 线程安全迭代预算管理 |
| `run_agent.py:7373` | `_invoke_tool()` | 工具调用拦截器（拦截 todo/memory/session_search/delegate_task） |
| `run_agent.py:8284` | `run_conversation()` | 完整对话循环入口 |
| `run_agent.py:2500` | `_persist_session()` | 会话持久化（JSONL + SQLite） |

### 7.3 工具系统

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `tools/registry.py:437` | `ToolRegistry` singleton | 工具注册表，单例模式 |
| `tools/registry.py:176` | `registry.register()` | 工具注册 API |
| `tools/registry.py:292` | `registry.dispatch()` | 工具执行分发 |
| `tools/registry.py:56` | `discover_builtin_tools()` | AST 扫描自动发现 |
| `model_tools.py:132` | `discover_builtin_tools()` | 触发所有 tools/*.py 导入 |
| `model_tools.py:196` | `get_tool_definitions()` | 返回过滤后的工具 schemas |
| `model_tools.py:421` | `handle_function_call()` | 主工具分发器 |
| `toolsets.py:31` | `_HERMES_CORE_TOOLS` | 核心工具列表定义 |
| `toolsets.py:447` | `resolve_toolset()` | 递归解析 toolset（含菱形依赖检测） |

### 7.4 委托系统

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `tools/delegate_tool.py:623` | `delegate_task()` | 委托入口，创建子 Agent |
| `tools/delegate_tool.py:238` | `_build_child_agent()` | 子 Agent 构建（含隔离配置） |
| `tools/delegate_tool.py:399` | `_run_single_child()` | 运行子 Agent（阻塞） |
| `tools/delegate_tool.py:53` | `MAX_DEPTH = 2` | 最大委托深度限制 |
| `tools/delegate_tool.py:741` | `ThreadPoolExecutor` | 批量委托并行执行 |
| `model_tools.py:157` | `_last_resolved_tool_names` | 全局工具名缓存（保护协议） |

### 7.5 记忆系统

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `hermes_state.py:115` | `SessionDB` | SQLite 会话存储 + FTS5 全文搜索 |
| `hermes_state.py:355` | `create_session()` | 创建会话记录 |
| `hermes_state.py:791` | `append_message()` | 追加消息到会话 |
| `hermes_state.py:990` | `search_messages()` | FTS5 全文搜索 |
| `tools/memory_tool.py:105` | `MemoryStore` | MEMORY.md/USER.md 文件记忆 |
| `tools/memory_tool.py:124` | `load_from_disk()` | 加载记忆文件（冻结快照） |
| `tools/memory_tool.py:359` | `format_for_system_prompt()` | 系统提示词格式化 |
| `tools/todo_tool.py:25` | `TodoStore` | 会话内 Todo 列表（内存） |
| `tools/todo_tool.py:156` | `todo_tool()` | Todo 工具入口 |
| `agent/memory_provider.py:42` | `MemoryProvider` ABC | 可插拔记忆提供者基类 |
| `agent/memory_manager.py:83` | `MemoryManager` | 协调内置 + 外部记忆提供者 |
| `agent/memory_manager.py:296` | `on_pre_compress()` | 压缩前通知各提供者 |
| `agent/context_compressor.py` | `ContextCompressor` | 长对话摘要压缩 |

### 7.6 执行后端

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `tools/environments/base.py:252` | `BaseEnvironment` ABC | 执行后端抽象基类 |
| `tools/environments/base.py:169` | `_ThreadedProcessHandle` | 阻塞 SDK 调用的线程包装 |
| `tools/environments/local.py:216` | `LocalEnvironment` | 本地主机执行 |
| `tools/environments/docker.py:235` | `DockerEnvironment` | Docker 容器执行 |
| `tools/environments/ssh.py:31` | `SSHEnvironment` | SSH 远程执行 |
| `tools/environments/singularity.py` | `SingularityEnvironment` | Singularity 容器 |
| `tools/environments/modal.py` | `ModalEnvironment` | Modal 云沙箱 |
| `tools/environments/daytona.py` | `DaytonaEnvironment` | Daytona 云沙箱 |
| `tools/terminal_tool.py:602` | `TERMINAL_ENV` | 环境类型选择 |

### 7.7 安全审批

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `tools/approval.py:76` | `DANGEROUS_PATTERNS` | 危险命令正则模式 |
| `tools/approval.py:694` | `check_all_command_guards()` | 主审批流程 |
| `tools/approval.py:535` | `_smart_approve()` | LLM 智能审批 |
| `tools/approval.py:377` | 持久化白名单 | `command_allowlist` |
| `tools/tirith_security.py` | `Tirith` | 二进制安全扫描器 |

### 7.8 网关与平台

| 文件 | 类/函数 | 核心职能 |
|------|---------|---------|
| `gateway/run.py` | `HermesGateway` | 主网关循环 + slash 命令 |
| `gateway/session.py` | `SessionStore` | 会话持久化（SQLite + JSONL） |
| `gateway/platforms/telegram.py` | `TelegramAdapter` | Telegram 平台适配器 |
| `gateway/platforms/discord.py` | `DiscordAdapter` | Discord 平台适配器 |
| `gateway/platforms/slack.py` | `SlackAdapter` | Slack 平台适配器 |

---

## 附录：关键行号索引

| 组件 | 文件:行号 |
|------|----------|
| 主循环条件 | `run_agent.py:8646` |
| API 调用 | `run_agent.py:8993` |
| 工具分发 | `run_agent.py:7373` (`_invoke_tool`) |
| 注册表分发 | `tools/registry.py:292` |
| 注册表发现 | `tools/registry.py:56` |
| 工具定义 | `model_tools.py:196` |
| 工具调用处理 | `model_tools.py:421` |
| 核心工具列表 | `toolsets.py:31` |
| 委托入口 | `tools/delegate_tool.py:623` |
| 子 Agent 构建 | `tools/delegate_tool.py:238` |
| 子 Agent 运行 | `tools/delegate_tool.py:399` |
| MAX_DEPTH | `tools/delegate_tool.py:53` |
| SessionDB | `hermes_state.py:115` |
| MemoryStore | `tools/memory_tool.py:105` |
| TodoStore | `tools/todo_tool.py:25` |
| 危险命令检查 | `tools/approval.py:694` |
| Docker 环境 | `tools/environments/docker.py:235` |
| Local 环境 | `tools/environments/local.py:216` |
| SSH 环境 | `tools/environments/ssh.py:31` |
| BaseEnvironment ABC | `tools/environments/base.py:252` |
| HermesCLI | `cli.py:1590` |
| AIAgent | `run_agent.py:535` |
| IterationBudget | `run_agent.py:659` |
| MemoryManager | `agent/memory_manager.py:83` |
| MemoryProvider ABC | `agent/memory_provider.py:42` |

---

*文档生成时间: 2026-04-21*
*分析深度: 源码级（行号追溯 + 交互流程还原）*

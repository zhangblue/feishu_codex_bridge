# 飞书 Codex 本地桥接实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 构建一个仅供指定飞书用户使用的 macOS 本地桥接服务，使其能够切换多个白名单项目、持续使用独立 Codex 会话，并通过一次性授权审批高风险操作。

**架构：** Python 常驻进程通过飞书官方长连接收发消息，以 SQLite 保存项目、会话、任务和审批状态，并用 `codex exec --json` 启动或恢复 Codex thread。Codex 插件提供安装/诊断 Skill，以及 PreToolUse、PermissionRequest、PostToolUse 策略 Hook；Hook 通过 Unix 域套接字联系桥接进程，用“首次拒绝并登记、飞书批准、恢复 thread 后单次放行”的方式适配非交互执行模式。

**技术栈：** Python 3.9+、`lark-oapi`、SQLite、TOML、pytest、Codex CLI 0.158+、Codex lifecycle hooks、macOS LaunchAgent。

---

## 文件结构

```text
.
├── pyproject.toml                         # Python 包、运行依赖、测试配置和 CLI 入口
├── .gitignore                             # 本地凭据、数据库、日志和虚拟环境排除规则
├── README.md                              # 安装、飞书配置、运行、审批和故障排查
├── config.example.toml                    # 无敏感信息的完整配置示例
├── src/feishu_codex_bridge/
│   ├── __init__.py                        # 包版本
│   ├── cli.py                             # run/check/install-launch-agent 等命令入口
│   ├── config.py                          # TOML 加载、环境变量和配置校验
│   ├── models.py                          # 项目、任务、事件和审批领域类型
│   ├── projects.py                        # 项目白名单与真实路径解析
│   ├── storage.py                         # SQLite schema、事务和状态仓储
│   ├── auth.py                            # 飞书 open_id 白名单校验
│   ├── commands.py                        # /projects、/use 等命令解析
│   ├── redaction.py                       # 日志和用户错误中的敏感信息清理
│   ├── codex_events.py                    # Codex JSONL 到统一事件的解析
│   ├── codex_runner.py                    # Codex 子进程、thread 恢复、中断和超时
│   ├── policy.py                          # 高风险工具调用的确定性识别规则
│   ├── approvals.py                       # 请求哈希、审批状态和一次性授权
│   ├── hook_server.py                     # Unix 域套接字审批协议服务端
│   ├── feishu.py                          # 飞书长连接、消息回复与审批卡片
│   ├── service.py                         # 应用编排、单任务锁和消息生命周期
│   └── launchd.py                         # LaunchAgent plist 生成、安装与卸载
├── plugins/feishu-codex-bridge/
│   ├── .codex-plugin/plugin.json          # Codex 兼容性插件清单
│   ├── hooks/hooks.json                   # 三类策略 Hook 注册
│   ├── hooks/hook_client.py               # 标准库实现的套接字客户端
│   ├── hooks/policy_hook.py               # Codex Hook 输入输出适配
│   └── skills/feishu-codex-bridge/SKILL.md # 安装、配置和诊断工作流
├── .agents/plugins/marketplace.json       # 仓库内插件目录条目
└── tests/
    ├── fixtures/
    │   ├── codex_success.jsonl            # 新会话成功事件样本
    │   ├── codex_resume.jsonl             # 恢复会话事件样本
    │   └── fake_codex.py                   # 可控的 Codex CLI 替身
    ├── test_config_projects.py
    ├── test_storage.py
    ├── test_auth_commands.py
    ├── test_codex_runner.py
    ├── test_approvals.py
    ├── test_hook_protocol.py
    ├── test_feishu_adapter.py
    ├── test_service.py
    ├── test_launchd.py
    └── test_plugin.py
```

## 任务 1：建立 Python 包、配置与项目白名单

**文件：**
- 创建：`pyproject.toml`
- 创建：`.gitignore`
- 创建：`config.example.toml`
- 创建：`src/feishu_codex_bridge/__init__.py`
- 创建：`src/feishu_codex_bridge/models.py`
- 创建：`src/feishu_codex_bridge/config.py`
- 创建：`src/feishu_codex_bridge/projects.py`
- 创建：`tests/test_config_projects.py`

- [ ] **步骤 1：编写配置和项目路径的失败测试**

```python
# tests/test_config_projects.py
from pathlib import Path

import pytest

from feishu_codex_bridge.config import ConfigError, load_config
from feishu_codex_bridge.projects import ProjectRegistry


def write_config(path: Path, project: Path) -> None:
    path.write_text(
        "[feishu]\n"
        'allowed_open_ids = ["ou_owner"]\n\n'
        "[projects.bridge]\n"
        f'path = "{project}"\n'
        "default = true\n\n"
        "[codex]\n"
        'sandbox = "workspace-write"\n'
        'approval_policy = "on-request"\n',
        encoding="utf-8",
    )


def test_loads_single_default_project(tmp_path, monkeypatch):
    project = tmp_path / "repo"
    project.mkdir()
    config_file = tmp_path / "config.toml"
    write_config(config_file, project)
    monkeypatch.setenv("FEISHU_APP_ID", "cli_test")
    monkeypatch.setenv("FEISHU_APP_SECRET", "secret")

    config = load_config(config_file)
    registry = ProjectRegistry(config.projects)

    assert config.feishu.allowed_open_ids == frozenset({"ou_owner"})
    assert registry.default.name == "bridge"
    assert registry.resolve("bridge").path == project.resolve()


def test_rejects_two_default_projects(tmp_path, monkeypatch):
    project_a = tmp_path / "repo-a"
    project_b = tmp_path / "repo-b"
    project_a.mkdir()
    project_b.mkdir()
    config_file = tmp_path / "config.toml"
    config_file.write_text(
        "[feishu]\n"
        'allowed_open_ids = ["ou_owner"]\n\n'
        "[projects.a]\n"
        f'path = "{project_a}"\n'
        "default = true\n\n"
        "[projects.b]\n"
        f'path = "{project_b}"\n'
        "default = true\n",
        encoding="utf-8",
    )
    monkeypatch.setenv("FEISHU_APP_ID", "cli_test")
    monkeypatch.setenv("FEISHU_APP_SECRET", "secret")

    with pytest.raises(ConfigError, match="exactly one default project"):
        load_config(config_file)
```

- [ ] **步骤 2：运行测试并确认导入失败**

运行：`python3 -m pytest tests/test_config_projects.py -q`

预期：FAIL，提示 `No module named 'feishu_codex_bridge'`。

- [ ] **步骤 3：建立包和依赖配置**

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "feishu-codex-bridge"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
  "lark-oapi>=1.5,<2",
  "tomli>=2; python_version < '3.11'",
]

[project.optional-dependencies]
dev = ["pytest>=8,<9"]

[project.scripts]
feishu-codex-bridge = "feishu_codex_bridge.cli:main"

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]

[tool.setuptools.packages.find]
where = ["src"]
```

在 `models.py` 定义不可变的 `ProjectConfig`、`FeishuConfig`、`CodexConfig` 和 `BridgeConfig` dataclass。`load_config()` 使用 Python 3.11 的 `tomllib`，在 Python 3.9/3.10 回退到 `tomli`；飞书密钥只从 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET` 读取。

`.gitignore` 必须排除 `.venv/`、`*.db`、`*.db-wal`、`*.db-shm`、`logs/`、`.env`、`config.toml`、`__pycache__/` 和 pytest 缓存，但保留 `config.example.toml`。

`ProjectRegistry` 在构造时调用 `Path.resolve(strict=True)`，拒绝非目录、重复真实路径、非法别名以及不是恰好一个默认项目的配置，并提供：

```python
class ProjectRegistry:
    @property
    def default(self) -> ProjectConfig: ...
    def list(self) -> tuple[ProjectConfig, ...]: ...
    def resolve(self, name: str) -> ProjectConfig: ...
```

- [ ] **步骤 4：补齐成功、缺失密钥、非法别名和不存在路径测试并运行**

运行：`python3 -m pytest tests/test_config_projects.py -q`

预期：PASS，所有项目路径均为真实绝对路径，错误信息不包含 App Secret。

- [ ] **步骤 5：提交配置基础**

```bash
git add pyproject.toml .gitignore config.example.toml src/feishu_codex_bridge tests/test_config_projects.py
git commit -m "feat: add bridge configuration and project registry"
```

## 任务 2：实现 SQLite 状态仓储和幂等事务

**文件：**
- 修改：`src/feishu_codex_bridge/models.py`
- 创建：`src/feishu_codex_bridge/storage.py`
- 创建：`tests/test_storage.py`

- [ ] **步骤 1：编写消息幂等、项目选择和 thread 映射的失败测试**

```python
# tests/test_storage.py
from feishu_codex_bridge.storage import StateStore


def test_claim_message_is_atomic(tmp_path):
    store = StateStore(tmp_path / "bridge.db")
    store.initialize(default_project="bridge")

    first = store.claim_message("om_123", sender_open_id="ou_owner")
    second = store.claim_message("om_123", sender_open_id="ou_owner")

    assert first.claimed is True
    assert second.claimed is False
    assert second.task_id == first.task_id


def test_project_threads_are_isolated(tmp_path):
    store = StateStore(tmp_path / "bridge.db")
    store.initialize(default_project="bridge")
    store.set_active_thread("bridge", "thread-a")
    store.set_active_thread("other", "thread-b")

    assert store.get_active_thread("bridge") == "thread-a"
    assert store.get_active_thread("other") == "thread-b"
```

- [ ] **步骤 2：运行测试并确认仓储不存在**

运行：`python3 -m pytest tests/test_storage.py -q`

预期：FAIL，提示无法导入 `StateStore`。

- [ ] **步骤 3：实现 schema 与显式事务**

创建表 `app_state`、`threads`、`messages`、`tasks` 和 `approvals`。连接启用：

```python
connection.execute("PRAGMA foreign_keys = ON")
connection.execute("PRAGMA journal_mode = WAL")
connection.row_factory = sqlite3.Row
```

提供窄接口：

```python
class StateStore:
    def initialize(self, default_project: str) -> None: ...
    def get_current_project(self) -> str: ...
    def set_current_project(self, project: str) -> None: ...
    def claim_message(self, message_id: str, sender_open_id: str) -> MessageClaim: ...
    def get_active_thread(self, project: str) -> str | None: ...
    def set_active_thread(self, project: str, thread_id: str) -> None: ...
    def create_task(self, message_id: str, project: str) -> TaskRecord: ...
    def finish_task(self, task_id: str, status: TaskStatus, error: str | None = None) -> None: ...
    def mark_running_tasks_interrupted(self) -> int: ...
```

Python 3.9 不支持 `X | None` 运行时注解时，在模块顶部启用 `from __future__ import annotations`。

- [ ] **步骤 4：增加重启中断与事务回滚测试并运行**

测试 `mark_running_tasks_interrupted()` 只修改 `running` 任务；故意触发唯一键冲突后确认当前项目未被部分更新。

运行：`python3 -m pytest tests/test_storage.py -q`

预期：PASS。

- [ ] **步骤 5：提交状态仓储**

```bash
git add src/feishu_codex_bridge/models.py src/feishu_codex_bridge/storage.py tests/test_storage.py
git commit -m "feat: persist bridge state in sqlite"
```

## 任务 3：实现身份校验、命令解析和敏感信息脱敏

**文件：**
- 创建：`src/feishu_codex_bridge/auth.py`
- 创建：`src/feishu_codex_bridge/commands.py`
- 创建：`src/feishu_codex_bridge/redaction.py`
- 创建：`tests/test_auth_commands.py`

- [ ] **步骤 1：编写白名单、命令和脱敏失败测试**

```python
# tests/test_auth_commands.py
import pytest

from feishu_codex_bridge.auth import UnauthorizedSender, require_owner
from feishu_codex_bridge.commands import CommandKind, parse_message
from feishu_codex_bridge.redaction import redact


def test_rejects_non_owner_without_detail():
    with pytest.raises(UnauthorizedSender, match="sender is not authorized"):
        require_owner("ou_other", frozenset({"ou_owner"}))


def test_parses_use_command_and_plain_prompt():
    command = parse_message("/use bridge")
    prompt = parse_message("修复失败的单元测试")

    assert command.kind is CommandKind.USE
    assert command.argument == "bridge"
    assert prompt.kind is CommandKind.PROMPT


def test_redacts_secret_bearer_and_app_secret():
    value = "Authorization: Bearer abc123 FEISHU_APP_SECRET=xyz"
    assert redact(value) == "Authorization: Bearer [REDACTED] FEISHU_APP_SECRET=[REDACTED]"
```

- [ ] **步骤 2：运行测试并确认模块不存在**

运行：`python3 -m pytest tests/test_auth_commands.py -q`

预期：FAIL，提示无法导入三个新模块。

- [ ] **步骤 3：实现明确的命令联合类型**

`CommandKind` 必须包含 `PROJECTS`、`USE`、`CURRENT`、`NEW`、`RESUME`、`STOP`、`HELP` 和 `PROMPT`。`parse_message()` 拒绝空消息、多余参数和未知斜杠命令，并将普通文本原样作为 prompt 保存。

`require_owner()` 使用集合精确匹配 `open_id`。`redact()` 至少清理 Bearer token、常见 secret/key/password 赋值和配置中的飞书 App Secret。

- [ ] **步骤 4：补充全部命令分支和空白输入测试并运行**

运行：`python3 -m pytest tests/test_auth_commands.py -q`

预期：PASS。

- [ ] **步骤 5：提交输入边界**

```bash
git add src/feishu_codex_bridge/auth.py src/feishu_codex_bridge/commands.py src/feishu_codex_bridge/redaction.py tests/test_auth_commands.py
git commit -m "feat: validate users commands and sensitive output"
```

## 任务 4：实现 Codex JSONL 事件解析和子进程运行器

**文件：**
- 创建：`src/feishu_codex_bridge/codex_events.py`
- 创建：`src/feishu_codex_bridge/codex_runner.py`
- 创建：`tests/fixtures/codex_success.jsonl`
- 创建：`tests/fixtures/codex_resume.jsonl`
- 创建：`tests/fixtures/fake_codex.py`
- 创建：`tests/test_codex_runner.py`

- [ ] **步骤 1：保存当前 Codex 0.158 JSONL 样本并编写解析失败测试**

```python
# tests/test_codex_runner.py
import asyncio
from pathlib import Path

from feishu_codex_bridge.codex_events import EventKind, parse_event
from feishu_codex_bridge.codex_runner import CodexRunner, RunRequest


def test_unknown_json_event_is_forward_compatible():
    event = parse_event('{"type":"future.event","payload":{"x":1}}')
    assert event.kind is EventKind.UNKNOWN
    assert event.raw_type == "future.event"


def test_runner_extracts_thread_and_final_message(tmp_path):
    runner = CodexRunner(
        executable=["python3", "tests/fixtures/fake_codex.py"],
        timeout_seconds=5,
    )
    result = asyncio.run(runner.run(RunRequest(cwd=tmp_path, prompt="hello")))

    assert result.thread_id == "00000000-0000-4000-8000-000000000123"
    assert result.final_message == "done"
    assert result.exit_code == 0
```

- [ ] **步骤 2：运行测试并确认事件模块不存在**

运行：`python3 -m pytest tests/test_codex_runner.py -q`

预期：FAIL，提示无法导入 `codex_events`。

- [ ] **步骤 3：实现事件模型和安全 argv 构造**

新会话必须使用参数数组而非 shell 字符串：

```python
[
    "codex", "exec", "--json", "--color", "never",
    "--sandbox", "workspace-write",
    "-c", 'approval_policy="on-request"',
    "-C", str(request.cwd), request.prompt,
]
```

恢复会话使用：

```python
[
    "codex", "exec", "resume", "--json", "--color", "never",
    "-c", 'approval_policy="on-request"',
    request.thread_id, request.prompt,
]
```

启动子进程时将 `cwd` 设置为已验证项目路径，并把 Hook 套接字地址写入 `FEISHU_CODEX_BRIDGE_SOCKET`。逐行读取 stdout、单独收集并脱敏 stderr；识别 thread、进度、最终消息和错误事件。未知事件产生 `UNKNOWN` 事件而不是异常。

- [ ] **步骤 4：实现超时、取消和非零退出测试**

让 `fake_codex.py` 根据 prompt 输出成功事件、等待、发送无效 JSON 或以非零状态退出。验证：

- 超时会终止子进程并返回 `TIMED_OUT`；
- `cancel()` 先发送终止信号，超时后强制结束；
- 无效 JSON 被记录为解析错误，不泄露原始密钥；
- 恢复命令携带指定 thread ID。

运行：`python3 -m pytest tests/test_codex_runner.py -q`

预期：PASS。

- [ ] **步骤 5：在本机运行只读 Codex 烟雾测试**

运行：`codex exec --json --sandbox read-only -C . "只回复 OK，不修改文件"`

预期：输出合法 JSONL、包含可持久化会话 ID、最终回复包含 `OK`，退出码为 0。若认证或网络不可用，将完整错误归类为环境阻塞，不修改单元测试预期。

- [ ] **步骤 6：提交 Codex 运行器**

```bash
git add src/feishu_codex_bridge/codex_events.py src/feishu_codex_bridge/codex_runner.py tests/fixtures tests/test_codex_runner.py
git commit -m "feat: run and resume Codex sessions"
```

## 任务 5：实现审批领域模型、一次性授权和 Hook 套接字协议

**文件：**
- 修改：`src/feishu_codex_bridge/models.py`
- 修改：`src/feishu_codex_bridge/storage.py`
- 创建：`src/feishu_codex_bridge/policy.py`
- 创建：`src/feishu_codex_bridge/approvals.py`
- 创建：`src/feishu_codex_bridge/hook_server.py`
- 创建：`tests/test_approvals.py`
- 创建：`tests/test_hook_protocol.py`

- [ ] **步骤 1：编写规范化哈希和单次消费失败测试**

```python
# tests/test_approvals.py
from datetime import datetime, timedelta, timezone

from feishu_codex_bridge.approvals import ApprovalManager, operation_digest
from feishu_codex_bridge.storage import StateStore


def test_grant_is_bound_to_one_turn_and_completed_once(tmp_path):
    store = StateStore(tmp_path / "bridge.db")
    store.initialize(default_project="bridge")
    manager = ApprovalManager(store)
    operation = {"tool_name": "Bash", "tool_input": {"command": "git push"}}
    digest = operation_digest(operation)
    approval = manager.request("bridge", "thread-1", "ou_owner", operation)
    manager.approve(approval.id, "ou_owner", expires_in=timedelta(minutes=2))

    assert manager.begin_execution("bridge", "thread-1", "turn-2", digest) is True
    assert manager.begin_execution("bridge", "thread-1", "turn-2", digest) is True
    assert manager.complete_execution("bridge", "thread-1", "turn-2", digest) is True
    assert manager.begin_execution("bridge", "thread-1", "turn-3", digest) is False
```

- [ ] **步骤 2：运行测试并确认审批管理器不存在**

运行：`python3 -m pytest tests/test_approvals.py -q`

预期：FAIL，提示无法导入 `ApprovalManager`。

- [ ] **步骤 3：实现确定性摘要与数据库原子消费**

`operation_digest()` 使用排序键、紧凑分隔符的 UTF-8 JSON，再计算 SHA-256。授权必须绑定 project、thread、tool name、完整 tool input 摘要、owner open_id、nonce、过期时间、执行 turn 和 consumed 时间。

`policy.py` 提供 `requires_pretool_approval(tool_name, tool_input, cwd)`。它明确识别递归或强制删除、`git reset --hard`、`git clean -f`、`git push`、发布/部署命令、`sudo`、`launchctl`、系统配置写入、钥匙串读取和已知的目录逃逸。无法安全解析且包含 shell 控制符的破坏性命令按高风险处理。普通测试、构建、格式化、项目内编辑和只读 Git 命令返回 false。

授权状态流转为 `pending -> approved -> executing -> consumed`。`begin_execution()` 将 approved 授权绑定到当前 turn 并进入 executing；同一 turn、同一摘要后续经过 PermissionRequest 时仍返回允许。不同 turn 不能复用 executing 授权。PostToolUse 调用 `complete_execution()` 后才进入 consumed；进程在执行中断开时，该授权不能在新 turn 使用，并由过期清理关闭。

最终消费使用单条带条件的 `UPDATE`：

```sql
UPDATE approvals
SET consumed_at = :now
WHERE id = :id
  AND status = 'executing'
  AND execution_turn_id = :turn_id
  AND consumed_at IS NULL
  AND expires_at > :now
```

只有 `rowcount == 1` 才返回允许。

- [ ] **步骤 4：编写 Unix 套接字协议失败测试**

```python
# tests/test_hook_protocol.py
import asyncio

from feishu_codex_bridge.hook_server import HookServer, request_hook_decision


def test_first_request_denies_and_registers_approval(tmp_path, approval_manager):
    socket_path = tmp_path / "hook.sock"
    server = HookServer(socket_path, approval_manager)

    async def scenario():
        await server.start()
        try:
            response = await request_hook_decision(socket_path, {
                "session_id": "thread-1",
                "cwd": "/repo",
                "hook_event_name": "PermissionRequest",
                "tool_name": "Bash",
                "tool_input": {"command": "git push"},
            })
            assert response["decision"] == "deny"
            assert response["approval_id"]
        finally:
            await server.close()

    asyncio.run(scenario())
```

- [ ] **步骤 5：实现一行一个 JSON 的本地协议**

协议只监听权限为 `0600` 的 Unix 域套接字。请求最大 64 KiB，必须在 2 秒内读完，只接受 `PreToolUse`、`PermissionRequest` 和 `PostToolUse`。PreToolUse 对常规操作返回 `continue`；高风险操作和 PermissionRequest 返回：

```json
{"decision":"allow","approval_id":"...","message":"approved for this turn"}
```

或：

```json
{"decision":"deny","approval_id":"...","message":"approval required"}
```

PostToolUse 成功登记后返回 `{"decision":"recorded"}`。套接字不存在、超时、JSON 无效或数据库不可用时，PreToolUse 与 PermissionRequest 返回拒绝；PostToolUse 记录本地错误但不能撤销已发生的副作用。服务启动前删除同一用户拥有的旧 socket 文件；拒绝符号链接或其他文件类型。

- [ ] **步骤 6：补充过期、重放、操作变化、错误用户和并发消费测试并运行**

同时验证危险规则会拦截 `git reset --hard`、`git clean -fd`、`git push`、递归删除和钥匙串读取，但不会拦截 `pytest`、`git diff`、`git status` 和项目内格式化。

运行：`python3 -m pytest tests/test_approvals.py tests/test_hook_protocol.py -q`

预期：PASS；两个不同 turn 的并发消费者中只有一个得到允许。

- [ ] **步骤 7：提交审批核心**

```bash
git add src/feishu_codex_bridge/models.py src/feishu_codex_bridge/storage.py src/feishu_codex_bridge/policy.py src/feishu_codex_bridge/approvals.py src/feishu_codex_bridge/hook_server.py tests/test_approvals.py tests/test_hook_protocol.py
git commit -m "feat: add one-time approval protocol"
```

## 任务 6：实现飞书长连接适配器和审批卡片

**文件：**
- 创建：`src/feishu_codex_bridge/feishu.py`
- 创建：`tests/test_feishu_adapter.py`

- [ ] **步骤 1：用传输替身编写消息映射失败测试**

```python
# tests/test_feishu_adapter.py
from feishu_codex_bridge.feishu import FeishuAdapter


def test_maps_private_text_message_and_ignores_group(fake_transport):
    received = []
    adapter = FeishuAdapter(fake_transport, on_message=received.append)

    adapter.handle_message_event({
        "sender": {"sender_id": {"open_id": "ou_owner"}},
        "message": {
            "message_id": "om_1",
            "chat_id": "oc_1",
            "chat_type": "p2p",
            "message_type": "text",
            "content": '{"text":"/projects"}',
        },
    })
    adapter.handle_message_event({
        "sender": {"sender_id": {"open_id": "ou_owner"}},
        "message": {"message_id": "om_2", "chat_type": "group", "message_type": "text"},
    })

    assert [message.message_id for message in received] == ["om_1"]
```

- [ ] **步骤 2：运行测试并确认飞书模块不存在**

运行：`python3 -m pytest tests/test_feishu_adapter.py -q`

预期：FAIL，提示无法导入 `FeishuAdapter`。

- [ ] **步骤 3：实现 SDK 隔离层**

使用 `lark.EventDispatcherHandler.builder()` 注册：

```python
handler = (
    lark.EventDispatcherHandler.builder("", "")
    .register_p2_im_message_receive_v1(on_message_event)
    .register_p2_card_action_trigger(on_card_action)
    .build()
)
ws_client = lark.ws.Client(app_id, app_secret, event_handler=handler)
```

SDK 回调只负责提取字段并投递到桥接服务事件循环，不在 SDK 回调线程中运行 Codex。出站方法封装为 `reply_text()`、`send_progress()`、`send_approval_card()` 和 `update_approval_card()`，业务层不引用 SDK builder 类型。

审批卡片 value 只包含不可猜测的 `approval_id` 和动作 `approve`/`deny`；回调处理仍须在服务端校验 open_id、状态和有效期。

- [ ] **步骤 4：补充非文本消息、恶意卡片、SDK 失败和重试测试**

验证群聊、非文本消息和缺失 open_id 不进入业务层；伪造 approval ID 返回通用失败；出站消息采用有限次数指数退避，达到上限后抛出脱敏异常。

运行：`python3 -m pytest tests/test_feishu_adapter.py -q`

预期：PASS。

- [ ] **步骤 5：提交飞书适配器**

```bash
git add src/feishu_codex_bridge/feishu.py tests/test_feishu_adapter.py
git commit -m "feat: connect Feishu messages and approval cards"
```

## 任务 7：实现应用编排、项目命令和单任务并发控制

**文件：**
- 创建：`src/feishu_codex_bridge/service.py`
- 创建：`tests/test_service.py`

- [ ] **步骤 1：编写端到端服务失败测试**

```python
# tests/test_service.py
import asyncio

from feishu_codex_bridge.service import BridgeService


def test_switches_projects_and_keeps_independent_threads(app_fixture):
    service: BridgeService = app_fixture.service

    async def scenario():
        await service.handle(app_fixture.message("om_1", "/use bridge"))
        await service.handle(app_fixture.message("om_2", "检查项目"))
        await service.handle(app_fixture.message("om_3", "/use other"))
        await service.handle(app_fixture.message("om_4", "检查项目"))

    asyncio.run(scenario())

    assert app_fixture.store.get_active_thread("bridge") == "thread-bridge"
    assert app_fixture.store.get_active_thread("other") == "thread-other"
```

- [ ] **步骤 2：运行测试并确认服务模块不存在**

运行：`python3 -m pytest tests/test_service.py -q`

预期：FAIL，提示无法导入 `BridgeService`。

- [ ] **步骤 3：实现消息生命周期编排**

`BridgeService.handle()` 必须按此顺序执行：身份校验、消息 claim、命令解析、当前项目解析、任务登记、即时确认、Codex 执行、thread 保存、任务结束和最终回复。

使用一个 `asyncio.Lock` 保证最多一个 Codex 任务运行。锁已占用时：

- `/stop` 立即请求取消；
- `/current` 和 `/help` 正常返回；
- 新 prompt 返回“当前任务执行中”，不创建第二个任务。

实现 `/projects`、`/use`、`/current`、`/new`、`/resume`、`/stop` 和 `/help`。`/resume` 列出当前项目最近 10 个 thread，并只接受列表中的 ID。

- [ ] **步骤 4：接入审批后的恢复执行**

收到批准卡片时：

1. `ApprovalManager.approve()` 创建一次性授权；
2. 读取原审批的 project、thread 和操作摘要；
3. 恢复同一 thread，发送固定提示：`用户已批准审批记录 <id> 中展示的原始操作。仅重试完全相同的操作；若参数变化，请重新请求授权。`；
4. PreToolUse 或 PermissionRequest 将授权绑定到当前 turn；同一工具调用经过两个 Hook 时复用同一授权；
5. PostToolUse 消费授权后更新卡片为“已批准并使用”；
6. 若未进入 PostToolUse，任务结束时将卡片更新为“执行中断，授权已关闭”。

拒绝时恢复同一 thread，明确告诉 Codex 用户拒绝该操作，不得使用等价命令绕过。

- [ ] **步骤 5：补充重复消息、忙碌状态、停止和审批分支测试**

运行：`python3 -m pytest tests/test_service.py -q`

预期：PASS；重复 `message_id` 不会使 fake runner 调用次数增加。

- [ ] **步骤 6：运行当前全部测试并提交**

运行：`python3 -m pytest -q`

预期：PASS。

```bash
git add src/feishu_codex_bridge/service.py tests/test_service.py
git commit -m "feat: orchestrate Feishu and Codex tasks"
```

## 任务 8：创建 Codex 插件和三阶段策略 Hook

**文件：**
- 创建：`plugins/feishu-codex-bridge/.codex-plugin/plugin.json`
- 创建：`plugins/feishu-codex-bridge/hooks/hooks.json`
- 创建：`plugins/feishu-codex-bridge/hooks/hook_client.py`
- 创建：`plugins/feishu-codex-bridge/hooks/policy_hook.py`
- 创建：`plugins/feishu-codex-bridge/skills/feishu-codex-bridge/SKILL.md`
- 创建：`.agents/plugins/marketplace.json`
- 创建：`tests/test_plugin.py`

- [ ] **步骤 1：使用 plugin-creator 生成仓库插件骨架**

运行：

```bash
python3 /Users/zhangdi/.codex/skills/.system/plugin-creator/scripts/create_basic_plugin.py feishu-codex-bridge --path plugins --marketplace-path .agents/plugins/marketplace.json --with-skills --with-hooks --with-marketplace
```

目标插件名为 `feishu-codex-bridge`，插件父目录为仓库的 `plugins/`，仓库 marketplace 为 `.agents/plugins/marketplace.json`。不要生成空 MCP 或 app 映射，因为该插件不提供 MCP Server。

预期生成 `.codex-plugin/plugin.json`、空 `skills/`、空 `hooks/` 和 marketplace 条目；插件名和外层目录名必须一致。

- [ ] **步骤 2：编写插件与 Hook 的失败测试**

```python
# tests/test_plugin.py
import json
import os
import subprocess
from pathlib import Path


PLUGIN = Path("plugins/feishu-codex-bridge")


def test_manifest_and_hook_registration_are_consistent():
    manifest = json.loads((PLUGIN / ".codex-plugin/plugin.json").read_text())
    hooks = json.loads((PLUGIN / "hooks/hooks.json").read_text())
    assert manifest["name"] == "feishu-codex-bridge"
    assert "hooks" not in manifest
    assert set(hooks["hooks"]) == {"PreToolUse", "PermissionRequest", "PostToolUse"}


def test_hook_fails_closed_without_socket():
    env = {**os.environ, "FEISHU_CODEX_BRIDGE_SOCKET": "/missing/socket"}
    completed = subprocess.run(
        ["python3", str(PLUGIN / "hooks/policy_hook.py")],
        input='{"hook_event_name":"PermissionRequest","tool_name":"Bash","tool_input":{"command":"git push"}}',
        text=True,
        capture_output=True,
        env=env,
        check=True,
    )
    output = json.loads(completed.stdout)
    assert output["hookSpecificOutput"]["decision"]["behavior"] == "deny"
```

- [ ] **步骤 3：实现 Hook 配置和标准库客户端**

兼容性 manifest 不声明 `hooks` 字段，由 Codex 默认发现 `hooks/hooks.json`，从而符合 plugin-creator 验证规则。该文件对全部工具注册三个同步 Hook，三个事件调用同一个适配脚本：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${PLUGIN_ROOT}/hooks/policy_hook.py",
            "timeout": 5,
            "statusMessage": "Checking local safety policy"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${PLUGIN_ROOT}/hooks/policy_hook.py",
            "timeout": 5,
            "statusMessage": "Checking Feishu approval"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${PLUGIN_ROOT}/hooks/policy_hook.py",
            "timeout": 5,
            "statusMessage": "Recording approved operation"
          }
        ]
      }
    ]
  }
}
```

Hook 从 stdin 读取不超过 64 KiB 的 JSON，把原始字段传给 Unix 套接字，并按事件转换为 Codex 官方结构。PermissionRequest 拒绝格式为：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "Approval required in Feishu."
    }
  }
}
```

PermissionRequest 允许时仅把 `behavior` 改为 `allow`。PreToolUse 的 `continue` 响应不输出任何内容；拒绝时返回 `permissionDecision: "deny"`；允许已批准操作时返回 `permissionDecision: "allow"` 并把原始 `tool_input` 原样放入 `updatedInput`。PostToolUse 只登记完成状态，不输出控制决定。

缺少环境变量、套接字错误、超时、无效响应或未知决定时，PreToolUse 与 PermissionRequest 必须拒绝；PostToolUse 只把脱敏错误写到 stderr。危险识别在桥接服务的 `policy.py` 中完成，插件脚本不复制规则。

- [ ] **步骤 4：编写安装与诊断 Skill**

`SKILL.md` 说明触发场景、配置检查、服务状态、Hook 信任要求、测试消息步骤和失败排查。Skill 只指导本地管理，不读取或输出 App Secret。

- [ ] **步骤 5：运行测试与插件验证器**

运行：

```bash
python3 -m pytest tests/test_plugin.py tests/test_hook_protocol.py -q
python3 /Users/zhangdi/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py plugins/feishu-codex-bridge
```

预期：测试 PASS，验证器报告插件有效。

- [ ] **步骤 6：人工信任检查**

从仓库 marketplace 安装插件，审查并信任 Hook 定义。在一个临时 Git 仓库中运行会触发审批的只读演练，确认未信任 Hook 时 Codex 不会静默放行，信任后 Hook 能联系本地桥接服务。

- [ ] **步骤 7：提交插件**

```bash
git add plugins/feishu-codex-bridge .agents/plugins/marketplace.json tests/test_plugin.py
git commit -m "feat: package Codex approval plugin"
```

## 任务 9：实现 CLI、运行检查与可选 LaunchAgent

**文件：**
- 创建：`src/feishu_codex_bridge/cli.py`
- 创建：`src/feishu_codex_bridge/launchd.py`
- 创建：`tests/test_launchd.py`

- [ ] **步骤 1：编写 plist 渲染和安全路径失败测试**

```python
# tests/test_launchd.py
import plistlib

from feishu_codex_bridge.launchd import render_plist


def test_plist_runs_as_current_user_without_root(tmp_path):
    payload = render_plist(
        executable="/opt/bridge/bin/feishu-codex-bridge",
        config_path=tmp_path / "config.toml",
        log_dir=tmp_path / "logs",
    )
    plist = plistlib.loads(payload)

    assert plist["Label"] == "com.local.feishu-codex-bridge"
    assert plist["RunAtLoad"] is True
    assert plist["KeepAlive"]["SuccessfulExit"] is False
    assert "UserName" not in plist
    assert plist["ProgramArguments"][-2:] == ["--config", str(tmp_path / "config.toml")]
```

- [ ] **步骤 2：运行测试并确认 LaunchAgent 模块不存在**

运行：`python3 -m pytest tests/test_launchd.py -q`

预期：FAIL，提示无法导入 `render_plist`。

- [ ] **步骤 3：实现 CLI 子命令**

提供：

```text
feishu-codex-bridge run --config PATH
feishu-codex-bridge check --config PATH
feishu-codex-bridge install-launch-agent --config PATH
feishu-codex-bridge uninstall-launch-agent
feishu-codex-bridge status
```

`check` 验证配置、项目目录、SQLite 可写目录、Codex CLI 版本、插件 Hook 文件和飞书密钥环境变量，但不打印密钥。`run` 初始化数据库、把遗留运行任务标为 interrupted、启动 HookServer，再建立飞书长连接。

LaunchAgent plist 写入当前用户的 `~/Library/LaunchAgents/com.local.feishu-codex-bridge.plist`；安装前若文件已存在且内容不同，先退出并提示显式使用 `--replace`。卸载只处理这个精确 label 和精确文件。

- [ ] **步骤 4：补充安装覆盖保护和 CLI check 测试**

所有测试使用临时 HOME，禁止写真实 `~/Library/LaunchAgents`。验证重复安装、内容冲突、卸载不存在服务和缺少 Codex 的错误信息。

运行：`python3 -m pytest tests/test_launchd.py tests/test_config_projects.py -q`

预期：PASS。

- [ ] **步骤 5：提交本地运行管理**

```bash
git add src/feishu_codex_bridge/cli.py src/feishu_codex_bridge/launchd.py tests/test_launchd.py
git commit -m "feat: add bridge CLI and launch agent support"
```

## 任务 10：完成文档、全量验证和真实验收

**文件：**
- 修改：`README.md`
- 修改：`config.example.toml`
- 修改：`pyproject.toml`

- [ ] **步骤 1：编写可从零执行的 README**

README 按以下顺序说明：

1. 功能与安全边界。
2. Python 3.9+、Codex CLI 0.158+ 和飞书自建应用前置条件。
3. 飞书后台启用机器人、长连接、接收消息事件和卡片回调所需步骤。
4. 获取本人 `open_id` 并加入白名单的方法。
5. 创建虚拟环境、安装 `.[dev]`、复制配置和设置两个飞书环境变量。
6. 注册多个项目并运行 `check`。
7. 前台启动与 `/projects`、`/use`、`/new`、`/resume`、`/stop` 示例。
8. 安装、审查和信任 Codex 插件 Hook。
9. 高风险审批的两阶段行为和失败关闭原则。
10. 可选 LaunchAgent 的安装、查看日志和卸载。
11. 常见故障：飞书断线、Codex 未登录、Hook 未信任、socket 不存在、审批过期。

- [ ] **步骤 2：运行静态与全量测试**

运行：

```bash
python3 -m compileall -q src plugins/feishu-codex-bridge/hooks
python3 -m pytest -q
git diff --check
```

预期：全部退出码为 0。

- [ ] **步骤 3：运行配置和插件验证**

运行：

```bash
feishu-codex-bridge check --config config.example.toml
python3 /Users/zhangdi/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py plugins/feishu-codex-bridge
```

预期：示例配置检查只报告示例路径/凭据需要替换，不出现 traceback；插件验证通过。

- [ ] **步骤 4：执行真实飞书验收清单**

- 本人私聊 `/projects` 能看到全部注册项目。
- `/use` 后普通任务在正确目录执行。
- 两个项目得到不同的 Codex thread。
- 重复投递相同消息不会重复启动任务。
- 长任务可被 `/stop` 中断。
- `git push` 首次被拒绝并生成审批卡片。
- 拒绝后 Codex 不尝试等价绕过。
- 批准后相同操作仅放行一次，重复点击无效。
- 重启桥接进程后当前项目和 thread 映射仍存在。
- 非白名单账号发送消息时不触发 Codex。

- [ ] **步骤 5：提交文档与发布前验证结果**

```bash
git add README.md config.example.toml pyproject.toml
git commit -m "docs: add setup and operations guide"
```

- [ ] **步骤 6：记录最终证据**

运行：

```bash
git status -sb
git log --oneline --decorate -12
python3 -m pytest -q
```

预期：工作区干净，任务 1 至任务 10 的提交均存在，全量测试通过。只有这些证据齐全后才声明 MVP 实现完成。

## 规格覆盖自检

- 架构和组件边界：任务 1 至任务 9。
- 飞书私聊、白名单和长连接：任务 3、6、7。
- 多项目和独立 thread：任务 1、2、4、7。
- 常规自动执行与单任务约束：任务 4、7。
- 高风险审批和一次性授权：任务 5、7、8。
- 配置、SQLite、凭据与脱敏：任务 1、2、3。
- 异常、超时、幂等与重启：任务 2、4、6、7、9。
- LaunchAgent：任务 9。
- 单元、集成、CLI 契约和真实验收：任务 1 至任务 10。
- 安装与运维文档：任务 8 至任务 10。

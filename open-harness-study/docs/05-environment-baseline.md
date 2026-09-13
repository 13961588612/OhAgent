# 环境与测试基线

> 用途：跨阶段复用的"事实快照"。s12（压缩与成本）、s19（可观测）、s20（测试/CI）都要回看本文件。
> 记录日期：2026-09-12 ｜ 记录方式：在 `.refs/OpenHarness` 内实测

## 一、版本矩阵

| 项 | 值 | 来源 |
|---|---|---|
| 上游仓库 | `HKUDS/OpenHarness` | — |
| commit | `9b2efd7` | `git rev-parse --short HEAD` |
| 版本号 | `0.1.9` | `pyproject.toml` / `oh --version` |
| Python | 3.12.13（uv 托管 `cpython-3.12-windows-x86_64-none`） | `uv run python -c ...` |
| 平台 | Windows / PowerShell | — |
| 构建后端 | hatchling（`requires-python >= 3.10`） | `pyproject.toml` |
| 关键依赖 | `anthropic` `openai` `rich` `textual` `typer` `pydantic` `mcp` `croniter` `lark-oapi` | `pyproject.toml` |

## 二、入口点

| 命令 | 目标 |
|---|---|
| `oh` / `openharness` / `openh` | `openharness.cli:app` |
| `ohmo` | `ohmo.cli:app` |

## 三、测试基线（s00 实测）

```powershell
uv run pytest -q -p no:cacheprovider
# 结果：收集阶段失败 1 个模块，整体中断
#   tests/test_mcp/test_http_flow.py
#   ModuleNotFoundError: No module named 'mcp.server.fastmcp'

uv run pytest -q -p no:cacheprovider --ignore=tests/test_mcp/test_http_flow.py
# 结果：25 failed, 1122 passed, 11 skipped, 12 warnings in 275.64s (0:04:35)
```

**基线数字：1122 passed / 25 failed / 11 skipped / 约 4 分 35 秒**（Windows，只读沙箱，无模型凭据）。

失败分布（按模块归类）：

| 领域 | 代表用例 | 疑似原因 |
|---|---|---|
| swarm 加锁 | `test_swarm/test_lockfile.py` | 用例假定 POSIX 语义 |
| 任务与子进程 | `test_tasks/test_manager.py` | 需要写入/重启子进程 |
| 团队生命周期 | `test_swarm/test_team_lifecycle.py` | `FileExistsError`，写文件受限 |
| cron 时区 | `test_services/test_cron.py` | 时区断言与环境设置不一致 |
| hooks 转义 | `test_hooks/test_executor.py` | shell 元字符转义在 Windows 行为不同 |
| 插件生命周期 | `test_plugins/test_lifecycle_flow.py` | 需要写入插件目录 |
| UI / bridge | `test_ui/test_modes.py`、`test_bridge/test_session_flow.py` | 终端与工作目录写入 |
| 其余 | `test_autopilot`、`test_ohmo`、`test_tools`、`test_engine` 各 1–4 个 | 多为写入/平台相关 |

> 注意：本基线是**环境基线**，不是"代码坏了"。s20 建立质量门禁时，需要先决定这批用例是"平台跳过"还是"必须修"。

## 四、已知不兼容

**`mcp 2.2.0` 与上游代码不兼容**

- 现象：`from mcp.server.fastmcp import FastMCP` 报 `ModuleNotFoundError`。
- 原因：`mcp` 2.x 把 `FastMCP` 重命名为 `MCPServer`（`mcp.server.mcpserver`），其余 API 亦有变更。
- 诱因：上游 `pyproject.toml` 只声明 `mcp>=1.0.0`，没有上界，`uv` 解析到了 2.x。
- 处置选项：
  1. 临时：`uv add "mcp<2"` 或 `uv run --with "mcp<2" ...`，让 s00 的测试基线能完整收集。
  2. 长期：跟踪上游是否适配 v2（属 s10 MCP 阶段内容）。

## 五、实验环境隔离约定

```powershell
$env:OPENHARNESS_CONFIG_DIR = "C:\code\OhAgent\open-harness-study\capstone\.local\config"
$env:OPENHARNESS_DATA_DIR   = "C:\code\OhAgent\open-harness-study\capstone\.local\data"
$env:OPENHARNESS_LOGS_DIR   = "C:\code\OhAgent\open-harness-study\capstone\.local\logs"
```

依据：`src/openharness/config/paths.py`（第 19 / 41 / 58 行附近）中三个环境变量优先级最高。

## 六、无凭据时的 dry-run 事实

`uv run oh --dry-run` 在完全无 key 的环境下输出：

- Readiness：`warning`（原因：运行时客户端解析失败 + 认证缺失），并给出 `next actions`。
- Resolved Settings 仍回落到内置默认：`profile claude-api` / `model claude-sonnet-4-6` / `max_turns 200` / `effort medium`。
- Validation：`system prompt chars: 7977`；`mcp config errors: 0`。
- Discovery：`plugins 0` / `skills 10` / `slash commands 69` / `built-in tools 39` / `mcp servers 0`。
- 不连外部 MCP，不调模型，不执行工具。

这组数字可作为后续阶段的"配置漂移检测"参照：技能/命令/工具数量变化时，说明扩展或上游版本发生了变化。
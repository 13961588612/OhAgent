# s00 · 环境搭建与源码地图

> 前置：无 ｜ 建议投入：6–8h ｜ 难度：★☆☆

## 一、学习目标

1. 从源码跑通 OpenHarness（`oh` / `ohmo`），建立可复现的实验环境。
2. 建立"能力 → 模块"的心智地图，能在 30 秒内定位任意功能的源码位置。
3. 掌握三种运行形态：交互 TUI、headless（`-p`）、安全预览（`--dry-run`）。
4. 建立测试基线，知道如何快速验证自己的改动没破坏上游行为。

## 二、知识点清单

- **仓库布局**：`src/openharness`（核心 39.3k 行）、`ohmo/`（个人智能体 App）、`frontend/terminal`（React 前端）、`autopilot-dashboard`（Vite 看板）、`tests/`（约 100 个模块）、`scripts/`、`.github/workflows`——见 `docs/04-source-index.md`
- **打包与依赖**：`pyproject.toml` 使用 hatchling；依赖含 `anthropic` `openai` `rich` `textual` `typer` `pydantic` `mcp` `croniter` `lark-oapi` 等；`dev` extra 含 pytest/ruff/mypy
- **入口点**：`oh` / `openharness` / `openh` → `openharness.cli:app`；`ohmo` → `ohmo.cli:app`
- **运行形态**：`oh`（TUI）、`oh -p "..."`（headless）、`--output-format json|stream-json`、`--dry-run`
- **dry-run 语义**：不调模型、不执行工具、不 spawn subagent、不连 MCP；但会解析 settings、auth、system prompt、skills、commands、tools、MCP 配置；输出 `ready/warning/blocked` 与 `next actions`
- **平台与能力探测**：`platforms.py`
- **通用基础设施**：`utils/fs.py`（原子写）、`utils/file_lock.py`、`utils/shell.py`、`utils/network_guard.py`

## 三、关键接口 / 符号

- CLI：`oh`、`oh setup`、`oh auth status`、`oh --dry-run`、`oh -p "..."`
- 入口函数：`openharness.cli:app`、`openharness.__main__`
- 路径：`config/paths.py` 的 `get_config_dir()` / `get_data_dir()` / `get_project_config_dir()`

## 四、动手实验

> **逐步执行手册：[`RUNBOOK.md`](RUNBOOK.md)** —— 15 个步骤，每个命令都给出完整格式、参数说明、预期输出、结果解读与对应知识点，按它从上到下执行即可。

命令速览：

```powershell
cd C:\code\OhAgent\.refs\OpenHarness
uv sync --extra dev
uv run oh --help                      # 把 help 输出归档到 notes/
uv run oh --dry-run                   # 观察 readiness 与 next actions
uv run oh -p "Explain this repository" --output-format json
uv run pytest -q                      # 记录基线：通过数 / 跳过数 / 耗时
python scripts/test_harness_features.py
```

实测修正（2026-09-12）：

- `uv run pytest -q` 会因环境内 `mcp 2.x` 与上游不兼容而在**收集阶段中断**，请改用 `--ignore=tests/test_mcp/test_http_flow.py`；完整基线为 1122 通过 / 25 失败 / 11 跳过。
- `python scripts/test_harness_features.py` 需要真实模型凭据（kimi），**s00 跳过**，留到 s02 之后执行。
- `uv run oh -p "..."` 在无凭据时会报 `No API key configured.` 并返回退出码 1；`--dry-run` 在同一状态下返回 0 并输出 warning 报告。

任务：
1. 统计 `src/openharness` 各模块的行数占比，找出 5 个"巨型文件"并解释原因。
2. 画一张"用户输入 → 输出"的端到端链路草图（先猜，后续阶段再修正）。

## 五、验收标准

- 能不看文档说出：核心/ohmo/前端/测试四个顶层目录各自的职责。
- `oh --dry-run` 能在**未配置任何 key** 的情况下给出可解释的 `blocked/warning` 结论。
- 能解释 `-p` 与 `--dry-run -p` 的行为差异（哪些环节被跳过）。
- 测试基线已记录到 `notes/s00-环境与源码地图.md`。

## 六、进阶思考

- 为什么上游把 `ohmo` 做成**独立 App 而非 core 的一个 mode**？这种切分对企业平台意味着什么？
- `--dry-run` 的"可解析但不执行"边界是如何划定的？如果让你扩展它支持"预估 token 成本"，需要动哪些模块？
- 39k 行里哪部分是**不可替代的 Harness 内核**，哪部分是可被企业替换的适配层？
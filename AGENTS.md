# Repository Guidelines

## 项目结构与模块组织

本仓库是 OpenHarness 的**研习工程**，不是上游源码仓库。

- `open-harness-study/stages/sNN-<topic>/`：23 个阶段，每阶段含 `README.md`（目标/验收）与可选 `RUNBOOK.md`（逐步命令手册）。
- `open-harness-study/docs/`：全局参考（架构、术语、CLI 与源码索引、环境基线）。
- `open-harness-study/notes/`：个人笔记，命名 `sNN-<主题>.md`。
- `open-harness-study/capstone/`：毕业项目工作区；`.local/` 放实验配置、数据与日志。
- `.refs/OpenHarness/`：上游只读快照（`9b2efd7` / v0.1.9）。**只读**：不在这里写作业、不改源码、不提交。

## 构建、测试与开发命令

除注明外，均在 `C:\code\OhAgent\.refs\OpenHarness` 下执行：

- `uv sync --extra dev`：创建/同步 `.venv` 并安装 dev 依赖。
- `uv run oh --help`、`uv run oh --version`：验证 CLI 与版本。
- `uv run oh --dry-run`：只解析不执行的环境体检。
- `uv run pytest -q -p no:cacheprovider --ignore=tests/test_mcp/test_http_flow.py`：测试基线；裸跑会在收集阶段中断（`mcp 2.x` 不兼容）。
- `uv run ruff check src tests scripts`：与上游 CI 一致的静态检查。

## 代码风格与命名约定

- Python 目标 `>=3.10`（实测 3.12.13），4 空格缩进；`ruff` 行宽 100、`target-version = "py311"`；`mypy` 可跑但非强制门禁。
- 命名：模块与函数 `snake_case`，类 `PascalCase`，常量 `UPPER_SNAKE_CASE`。
- Markdown 文件用 `sNN-<主题>.md`；文档与笔记使用简体中文，代码注释也用中文。

## 测试指南

- 框架为 `pytest`（`asyncio_mode = "auto"`，`testpaths = ["tests"]`）；测试按模块镜像组织在 `tests/test_<module>/`。
- 文件命名 `test_*.py`，用例命名 `test_*`；新增用例放入对应模块目录。
- 当前基线：1122 passed / 25 failed / 11 skipped（Windows、无凭据）。这批失败多为平台与写盘差异，**不要顺手修复**，记录结论即可。

## 提交与 PR 指南

- 本工作区自身不是 git 仓库；唯一仓库是只读快照 `.refs/OpenHarness`，禁止在其中提交改动。
- 上游使用 Conventional Commits，如 `fix(config): preserve profile auth when overriding model`；提 PR 用 `<type>(<scope>): <描述>`。
- PR 要求：范围小而聚焦；说明问题、改动与验证方式；行为变更补测试；CLI 变更同步更新文档与 `CHANGELOG.md` 的 `Unreleased`。

## 实验与安全约定

- 实验前先隔离环境，避免污染个人 `~/.openharness`：
  `$env:OPENHARNESS_CONFIG_DIR = "$PWD\capstone\.local\config"`（`DATA_DIR`、`LOGS_DIR` 同理）；环境变量仅对当前 PowerShell 窗口有效。
- 密钥只通过环境变量注入，禁止提交到仓库；`--dry-run` 与 `config show` 输出应脱敏后再贴进笔记。
- 每步实验在 `notes/` 记录三样东西：命令原文、原始输出、你的意外之处。
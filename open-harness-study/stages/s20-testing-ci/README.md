# s20 · 测试、质量与 CI

> 前置：s19 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握上游测试组织方式，能按模块快速找到"该改哪里、该测什么"。
2. 掌握端到端验证脚本的用法，能自建质量门禁。
3. 能为自己的扩展（工具/技能/插件/钩子）建立回归测试。

## 二、知识点清单

- **测试组织**（`tests/`，约 100 个模块，按源码模块镜像）
  - `test_swarm/*`（12 个）：协作运行时覆盖最全
  - `test_tools/*`：bash/grep/mcp/任务/图像/web fetch
  - `test_services/*`：compact、cron、cron_scheduler、session_storage、autodream
  - `test_ui/*`：TUI、React 后端、权限模式、插件安全
  - `test_channels/*`：通道基础 + 飞书/Telegram 安全用例
  - `test_mcp/*`：stdio/http/integration/错误路径
  - `test_hooks/*`：executor、priority
  - `test_permissions/*`、`test_memory/*`、`test_prompts/*`、`test_sandbox/*`、`test_swarm/*`
  - `tests/fixtures/fake_mcp_server.py`：MCP 夹具
  - `tests/conftest.py`：公共夹具
- **端到端脚本**（`scripts/`）：`e2e_smoke.py`、`test_harness_features.py`、`test_real_skills_plugins.py`、`test_cli_flags.py`、`test_headless_rendering.py`、`test_tui_interactions.py`、`react_tui_e2e.py`、`test_docker_sandbox_e2e.py`、`local_system_scenarios.py`
- **CI**（`.github/workflows/`）：`ci.yml`、`autopilot-scan.yml`、`autopilot-run-next.yml`、`autopilot-pages.yml`
- **工程配置**：`pytest`（`asyncio_mode=auto`、`testpaths=tests`）、`ruff`（line-length 100、py311）、`mypy`（strict）
- **规则**：新增能力必须带测试；改动公共契约（工具名、协议字段、配置字段）必须评估兼容性

## 三、关键接口 / 符号

- `uv run pytest -q`、`uv run pytest tests/test_tools -q`
- `uv run ruff check`、`uv run mypy src`
- 脚本：`python scripts/test_harness_features.py`、`python scripts/e2e_smoke.py`

## 四、动手实验

```powershell
uv run pytest -q
uv run pytest tests/test_swarm -q
uv run pytest tests/test_mcp -q
uv run ruff check src tests
uv run mypy src
python scripts/e2e_smoke.py
python scripts/test_cli_flags.py
```

任务：
1. 为 s05 的自定义工具补齐单测（正常、异常、超时、权限被拒四类）。
2. 为 s08 的企业技能写"命中率"回归用例（用多个真实提问验证描述是否能被选中）。
3. 写一个 CI 门禁脚本：`pytest + ruff + 关键 e2e`，任一失败即阻断。
4. 制造一次"工具名从 snake_case 改成 camelCase"的破坏性改动，观察哪些测试会红——理解契约的连锁影响。

## 五、验收标准

- 能说出"改某模块时该跑哪些测试"。
- 自研扩展有覆盖四类场景的测试，且能稳定通过。
- 能解释为什么 `asyncio_mode=auto` 对本项目重要（大量异步工具与通道）。
- 有一套可执行的质量门禁（脚本或 CI 配置）。

## 六、进阶思考

- 静态检查 `mypy strict` 在 39k 行规模下如何落地而不拖慢开发？
- 端到端脚本（真实模型调用）成本高、不稳定；如何设计"录制回放"或 mock 层？
- 升级上游版本时，如何用测试快速定位破坏性变更？
- 企业自研扩展的测试应该与上游测试分离还是合并？如何避免上游升级带来的维护地狱？
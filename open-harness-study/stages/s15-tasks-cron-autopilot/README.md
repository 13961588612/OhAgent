# s15 · 后台任务 / Cron / Autopilot

> 前置：s14 ｜ 建议投入：10–12h ｜ 难度：★★★★

## 一、学习目标

1. 掌握后台任务的创建、查询、中止与输出读取。
2. 掌握定时（cron）能力与调度实现。
3. 理解 Autopilot 的"策略 → 运行 → 验证 → 发布"闭环，并能改造为企业的自动化流水线。

## 二、知识点清单

- **后台任务**（`tasks/`）
  - `manager.py`（约 19 KB）：任务生命周期与输出管理
  - `types.py`：任务模型；`local_agent_task.py`：本地 Agent 任务；`local_shell_task.py`：本地 shell 任务；`stop_task.py`：中止
  - 工具面：`task_create` `task_get` `task_list` `task_stop` `task_output` `task_update`
  - 存储：`~/.openharness/data/tasks/`
- **Cron**
  - `services/cron.py`：任务定义与管理
  - `services/cron_scheduler.py`（约 20 KB）：调度（依赖 `croniter`）
  - 存储：`~/.openharness/data/cron_jobs.json`
  - CLI：`oh cron start|stop|status|list|toggle|history|logs`
  - 工具：`cron_create` `cron_list` `cron_delete` `cron_toggle`
  - 按需触发：`remote_trigger` 工具 + `/remote-trigger` 语义
- **Autopilot**（`autopilot/service.py`，约 95 KB——本项目最大单文件之一）
  - 项目级状态：`.openharness/autopilot/`（`registry.json`、`repo_journal.jsonl`、`active_repo_context.md`、`runs/`）
  - 策略文件：`autopilot_policy.yaml`、`verification_policy.yaml`、`release_policy.yaml`
  - CLI：`oh autopilot status|list|add|context|journal|scan|run-next|tick|install-cron|export-dashboard`
  - 与 GitHub 集成：`.github/workflows/autopilot-*.yml`，看板 `autopilot-dashboard`、静态站 `docs/autopilot/`
  - 概念：issue 接收（`.openharness/issue.md`）、PR 评论（`pr_comments.md`）、验证、发布
- **测试参考**：`tests/test_services/test_cron*.py`、`tests/test_autopilot/test_verification.py`、`tests/test_tasks/test_manager.py`

## 三、关键接口 / 符号

- 任务工具六件套：`task_create/get/list/stop/output/update`
- `oh cron *`、`/tasks`
- Autopilot 策略：`autopilot_policy.yaml` / `verification_policy.yaml` / `release_policy.yaml`

## 四、动手实验

```powershell
uv run oh -p "创建一个后台任务：跑完整测试套件并把失败用例写进 report.md"
uv run oh -p "/tasks"

uv run oh cron start
uv run oh cron list
uv run oh cron add --help 2>$null ; uv run oh cron status

uv run oh autopilot status
uv run oh autopilot scan
uv run oh autopilot context
uv run oh autopilot run-next
```

任务：
1. 起一个长耗时后台任务，期间继续对话，验证任务互不干扰；再读取其输出并中止它。
2. 配置一个每日 cron 任务（如"每天 9 点汇总仓库 TODO 并发到飞书"），验证触发与日志。
3. 读 `autopilot/service.py` 的策略加载部分，写一份最小的 `autopilot_policy.yaml` 并跑 `scan` + `run-next`。
4. 设计企业版自动化流水线：需求接收 → 编码 → 自测 → 验证门禁 → 发布候选 → 人工审批。

## 五、验收标准

- 能说明后台任务与"同步工具调用"的区别，以及在什么场景必须用后台。
- 能解释 cron 的持久化与调度方式，以及进程重启后的行为。
- 能说清 Autopilot 三类策略文件各自的职责与相互关系。
- 能给出"无人值守"的安全前提（权限模式、沙箱、审计、失败回滚）。

## 六、进阶思考

- Autopilot 追求"自动改代码并验证"。企业要落地，最大的阻碍是**信任**还是**能力**？如何用门禁与回滚换取信任？
- Cron 是单机调度；多副本部署时会重复执行吗？需要什么（分布式锁、幂等、主选举）？
- 后台任务输出可能极大，与企业日志/对象存储如何集成？
- 无人值守下的"卡住"如何检测与自愈（超时、心跳、看门狗）？
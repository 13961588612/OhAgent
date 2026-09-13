# s19 · 可观测性与运营

> 前置：s18 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 梳理 OpenHarness **自带**的可观测能力（日志、用量、成本、会话、反馈）。
2. 明确企业场景的缺口，并设计补齐方案（OTel / 监控 / 看板）。
3. 能定位四类典型事故：越权、上下文爆炸、成本飙升、通道丢消息。

## 二、知识点清单

- **内置可观测面**
  - 日志目录：`~/.openharness/logs/`（`OPENHARNESS_LOGS_DIR`）
  - 用量与成本：`engine/cost_tracker.py`、`api/usage.py`、`/cost` `/usage` `/stats`
  - 会话与转录：`services/session_storage.py`、`/session` `/export` `/share`
  - 反馈：`~/.openharness/data/feedback/feedback.log`、`/feedback`
  - 诊断：`/doctor`、`oh --dry-run`（配置就绪度）、`/context`（提示词实况）
  - 运行证据：Autopilot 的 `runs/` 与 `repo_journal.jsonl`
- **可观测缺口（企业需补齐）**
  - 无统一 trace/span（跨轮次、跨工具、跨子 Agent）
  - 无指标导出（Prometheus/OTel metrics）
  - 审计日志需要自己做（结合 s07 的 `http` 钩子）
  - 成本归集到"团队/项目/用户"维度需要自研
- **落点设计参考**
  - `pre_tool_use` / `post_tool_use` → 审计与耗时指标
  - `pre_compact` / `post_compact` → 上下文压力指标
  - `session_start` / `session_end` → 会话与用户维度
  - `stop` / `subagent_stop` → 异常终止监控
- **运营指标建议**：成功率、平均轮次、工具调用分布、P95 时延、token/任务、成本/用户、压缩频次、记忆命中率、通道送达率

## 三、关键接口 / 符号

- `/doctor`、`/cost`、`/usage`、`/stats`、`/session`、`/export`
- 钩子事件（s07）作为埋点入口
- `autopilot/runs/`、`repo_journal.jsonl` 作为运行档案

## 四、动手实验

```powershell
uv run oh -p "/doctor"
uv run oh -p "/cost"
uv run oh -p "/usage"
uv run oh -p "/stats"
```

任务：
1. 用钩子搭一个**最小可观测栈**：`post_tool_use` → 本地 HTTP 服务 → JSONL → 脚本聚合出"工具调用 Top10 / 平均耗时"。
2. 定义并实现 5 个核心指标的计算脚本（可用示例数据）。
3. 写一份"事故手册"：四类事故各自的定位步骤与修复动作（越权、上下文爆炸、成本飙升、通道丢消息）。
4. 设计成本看板字段（按 profile / 团队 / 会话三个维度）。

## 五、验收标准

- 能列出 OpenHarness 自带能力与缺口的清单（各至少 5 条）。
- 埋点方案可运行，能产出真实统计数据。
- 事故手册中的每条定位路径都指向具体源码或命令。
- 能说明"为什么审计日志不能只依赖应用层日志"。

## 六、进阶思考

- 全链路 trace 需要贯穿子进程与队友（跨进程上下文传播）。设计上要解决什么？
- 采样策略如何定？（全量审计 vs 抽样指标）
- 成本与质量的权衡：如何用数据回答"该不该给这个团队加额度"？
- 隐私合规下，会话内容能否上报？脱敏应该在哪个层做（钩子 / 网关 / 存储）？
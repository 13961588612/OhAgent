# s12 · 上下文压缩与成本控制

> 前置：s11 ｜ 建议投入：8–10h ｜ 难度：★★★★

## 一、学习目标

1. 掌握压缩（compact）的触发条件、策略与产物，理解"什么时候会丢信息"。
2. 掌握 token 估算与成本统计链路，能建立企业级成本模型。
3. 能设计"长周期任务不爆上下文、不被账单击穿"的方案。

## 二、知识点清单

- **压缩服务**（`services/compact/__init__.py`，约 67 KB）
  - 自动压缩阈值：`auto_compact_threshold_tokens`
  - 上下文窗口：`context_window_tokens`（与 profile 绑定）
  - 流式压缩：循环内 `_stream_compaction`
  - 上下文塌缩（context collapse）：裁剪陈旧工具结果
  - 图像块处理：估算时计图像、摘要请求剥离图像负载
  - 抖动/溢出识别：可识别 llama.cpp / OpenAI 兼容后端的上溢错误
- **工具输出治理**：`services/tool_outputs.py` —— 超大输出落盘到 `tool_artifacts` 并在历史中保留引用；旧 MCP 结果可被 microcompact
- **Token 估算**：`services/token_estimation.py`
- **成本**：`engine/cost_tracker.py`、`api/usage.py`；`/cost`、`/usage`、`/stats`
- **压缩事件钩子**：`pre_compact`、`post_compact`（可在压缩前后做留痕或注入）
- **用户操作**：`/compact`、`/summary`
- **配置联动**：`MemorySettings.context_window_tokens` / `auto_compact_threshold_tokens`、`ProviderProfile` 同名字段

## 三、关键接口 / 符号

- `auto_compact_threshold_tokens`、`context_window_tokens`
- `pre_compact` / `post_compact` HookEvent
- `/compact`、`/cost`、`/usage`、`/stats`

## 四、动手实验

```powershell
uv run oh -p "/cost"
uv run oh -p "/usage"
uv run oh -p "/stats"
uv run oh -p "/compact"
uv run oh -p "把下面这段超长日志逐行分析一遍：<粘贴 5 万行>"   # 观察工具输出落盘与压缩
```

任务：
1. 把 `auto_compact_threshold_tokens` 调到很小，跑一段多轮对话，观察压缩触发点与信息保真度。
2. 用 `pre_compact` / `post_compact` 钩子记录每次压缩前后的 token 估算，画出曲线。
3. 对比 `/cost` 与 `/stats` 的口径差异，说明各自的数据来源。
4. 为一类真实任务（如"全仓库重构"）做成本预算：轮次 × 每轮 prompt/输出 token × 单价。

## 五、验收标准

- 能解释压缩后"哪些信息一定保留、哪些可能丢失"，并给出避免丢失的做法（外部化到文件/记忆）。
- 能读懂 token 估算的三个组成部分（系统提示、历史消息、工具 schema/结果）。
- 能给出企业级成本护栏方案（阈值告警、单会话上限、按团队配额）。
- 压缩前后钩子均能稳定触发并留痕。

## 六、进阶思考

- 摘要压缩会引入"二次幻觉"。如何评估压缩保真度（回归用例集）？
- 工具输出落盘后，模型只看到引用；什么情况下会导致模型"看不见关键信息"从而反复调用同一工具？
- 上下文窗口来自 profile 配置，若配置与实际模型不符会发生什么？如何做自动探测？
- 成本控制应该在 Harness 层还是网关层？各自的优缺点是什么？
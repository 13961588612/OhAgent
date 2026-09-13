# s04 · Agent Loop 引擎（核心必读）

> 前置：s03 ｜ 建议投入：10–12h ｜ 难度：★★★★

**本阶段是全路线的心脏。** 没读透 `engine/query.py`，后面所有治理与扩展都只能"照着配"。

## 一、学习目标

1. 完整复述 `run_query` 的控制流：模型调用 → 工具调用 → 结果回填 → 再调用，直到终止。
2. 理解 `QueryContext` 携带了哪些运行时状态，以及各治理层在哪些"插点"介入。
3. 掌握流式事件模型（`stream_events.py`），知道 TUI / headless / channel 三种消费者如何共用。
4. 理解终止条件：`max_turns`、`MaxTurnsExceeded`、用户中断与 `/continue`。

## 二、知识点清单

- **核心循环**（`engine/query.py`，约 40 KB）
  - `run_query(...)`：主循环
  - `QueryContext`：工具注册表、权限、设置、会话状态、钩子执行器、成本跟踪
  - `_execute_tool_call(...)`：单个工具调用的完整生命周期（权限 → pre hook → 执行 → post hook → 结果封装）
  - `_preprocess_images_in_messages(...)`：多模态消息预处理
  - `_stream_compaction(...)`：循环内的流式压缩
  - `MaxTurnsExceeded`：轮次护栏
- **高层引擎**（`engine/query_engine.py`）：会话级封装，供 UI/通道复用
- **消息模型**（`engine/messages.py`）：文本块、工具调用块、工具结果块、图像块等
- **流式事件**（`engine/stream_events.py`）：逐 token 文本、工具开始/结束、回合结束等
- **成本**（`engine/cost_tracker.py`）：usage 聚合
- **运行时装配**（`ui/runtime.py:build_runtime`）：把 settings/auth/tools/skills/plugins/mcp/hooks 组装成可运行上下文
- **运行参数**：`max_turns`（默认 200）、`effort`、`passes`、`fast_mode`——对应 `/turns` `/effort` `/passes` `/fast`
- **中断与恢复**：`/stop`、`/continue`、`/rewind`

## 三、关键接口 / 符号

- `run_query(...)`、`QueryContext`、`_execute_tool_call(...)`、`MaxTurnsExceeded`
- `StreamEvent`（`engine/stream_events.py`）
- `build_runtime(...)`（`ui/runtime.py`）
- 关键参数：`max_turns`、`permission mode`、`cwd`

## 四、动手实验

```powershell
uv run oh -p "List all functions in main.py" --output-format stream-json
uv run oh -p "/turns 3" ; uv run oh -p "做一个需要 10 步的任务"   # 观察护栏触发
uv run oh -p "/passes 2"
```

任务：
1. 画出 `run_query` 的伪代码（含所有分支：无工具调用、多工具并行、工具报错、轮次超限、被 hook 阻断）。
2. 找到"权限检查"与"pre_tool_use hook"的**先后顺序**，说明为什么是这个顺序。
3. 统计一次真实会话触发了几次模型调用、几次工具调用，与 `/cost` `/stats` 的输出对照。
4. 用 `stream-json` 输出还原一次工具调用事件的完整序列。

## 五、验收标准

- 能口头复述循环，并指出至少 6 个"治理插点"及其源码位置。
- 能解释 `max_turns` 用尽时的行为，以及如何安全地继续。
- 能说明 `build_runtime` 装配顺序，以及少了某个组件会在哪一步失败。
- 能读懂 `stream-json` 每一类事件字段。

## 六、进阶思考

- 循环是"串行工具执行"还是"可并行"？并行时权限确认如何串行化（TUI 提示：`_ask_permission` 用锁）？
- 如果一次工具调用需要 30 分钟（长构建），当前设计会发生什么？你会怎么改造（结合 s15 后台任务）？
- 上下文压缩在循环内"流式"触发，比"到阈值才压缩"好在哪？代价是什么？
- 这套循环与企业常见的 DAG/工作流引擎相比，优势与劣势分别是什么？
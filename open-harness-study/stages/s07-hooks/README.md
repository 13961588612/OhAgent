# s07 · Hooks 事件体系与合规

> 前置：s06 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握 10 个 HookEvent 的触发时机与语义。
2. 掌握 4 类 Hook（command / prompt / http / agent）的能力边界与代价。
3. 能落地企业级合规钩子：审计留痕、危险操作拦截、通知与阻断。

## 二、知识点清单

- **事件模型**（`hooks/events.py`）
  - 会话：`session_start`、`session_end`
  - 压缩：`pre_compact`、`post_compact`
  - 工具：`pre_tool_use`、`post_tool_use`
  - 交互：`user_prompt_submit`、`notification`
  - 终止：`stop`、`subagent_stop`
- **钩子类型**（`hooks/schemas.py`）
  - `command`：执行 shell 命令，`block_on_failure` 默认 `false`
  - `prompt`：让模型判断条件，默认 `true` 阻断
  - `http`：POST 事件负载到 HTTP 端点（对接审计平台）
  - `agent`：更深度的模型校验，默认 `true` 阻断
  - 通用字段：`matcher`（匹配条件）、`priority`（同事件内高优先先执行）、`timeout_seconds`
- **执行引擎**（`hooks/executor.py`）：逐个执行 → 聚合 `AggregatedHookResult` → 任一 `blocked` 即阻断并给出 `reason`
- **装载与热更新**：`hooks/loader.py`、`hooks/hot_reload.py`（settings 变更后尽力热重载）
- **安全细节**：命令钩子中的 `$ARGUMENTS` 会被 shell 转义（防注入）
- **可观察性**：`/hooks` 查看已配置钩子

## 三、关键接口 / 符号

- `HookEvent.{SESSION_START, SESSION_END, PRE_COMPACT, POST_COMPACT, PRE_TOOL_USE, POST_TOOL_USE, USER_PROMPT_SUBMIT, NOTIFICATION, STOP, SUBAGENT_STOP}`
- `HookResult{success, output, blocked, reason, metadata}`、`AggregatedHookResult.blocked/reason`
- `priority`（同事件内排序）、`matcher`、`block_on_failure`

## 四、动手实验

```powershell
uv run oh -p "/hooks"
uv run oh -p "/config"
```

任务：
1. 配置一个 `command` 钩子在 `post_tool_use` 记录审计日志（工具名、参数摘要、时间戳、会话 ID）。
2. 配置一个 `http` 钩子把 `pre_tool_use` 事件推送到本地 mock 审计服务。
3. 配置一个 `prompt` 钩子，在 `user_prompt_submit` 拦截包含敏感信息的请求。
4. 用 `priority` 让"安全检查"先于"日志记录"执行，并验证顺序。
5. 制造一次钩子失败，观察 `block_on_failure` 对流程的影响。

## 五、验收标准

- 能列出 10 个事件并说明每个事件适合做什么（留痕/拦截/通知/校验）。
- 能解释"钩子阻断"与"权限拒绝"的区别与配合方式。
- 产出一套可用的企业合规钩子配置（审计 + 拦截 + 通知），并有运行证据。
- 能说出 `agent` 类钩子的成本与适用边界（不要滥用）。

## 六、进阶思考

- `http` 钩子失败时是否应阻断主流程？企业审计要求"必达"时如何设计（本地缓冲 + 重试）？
- 钩子是同步的，长耗时校验会拖慢每一步。如何做到"关键路径同步、非关键路径异步"？
- 钩子配置散落在 `settings.json` 与插件中，如何做统一治理与版本管理？
- 若要求"所有工具调用必须留痕且不可被绕过"，钩子可被禁用吗？如何防止（结合插件信任与配置下发）？
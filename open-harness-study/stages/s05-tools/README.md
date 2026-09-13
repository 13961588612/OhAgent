# s05 · Tools 体系与自定义工具开发

> 前置：s04 ｜ 建议投入：10–12h ｜ 难度：★★★

## 一、学习目标

1. 掌握工具契约（`BaseTool` / `ToolRegistry` / `ToolExecutionContext` / `ToolResult`）。
2. 通读 43 个内置工具，形成"哪类需求用哪个工具"的索引能力。
3. 独立开发、注册、测试一个自定义工具，并让它被模型正确调用。

## 二、知识点清单

- **契约**（`tools/base.py`）：工具名、描述、参数 schema、执行入口、结果封装、错误语义
- **注册**（`tools/__init__.py:create_default_tool_registry`）：默认工具集 + MCP 动态适配器（`McpToolAdapter`）
- **内置工具分类**
  - 文件系统：`file_read` `file_write` `file_edit` `notebook_edit` `glob` `grep`
  - 执行与环境：`bash` `sleep` `enter_worktree` `exit_worktree`
  - 代码智能：`lsp`
  - 检索与网络：`web_search` `web_fetch` `tool_search`
  - 多模态：`image_generation` `image_to_text`
  - 交互与协作：`ask_user_question` `send_message` `brief`
  - 规划与状态：`enter_plan_mode` `exit_plan_mode` `todo_write` `config`
  - 技能：`skill`
  - MCP：`mcp_tool`（适配） `list_mcp_resources` `read_mcp_resource` `mcp_auth`
  - 任务与团队：`task_create/get/list/stop/output/update`、`team_create/delete`、`agent`
  - 定时与触发：`cron_create/list/delete/toggle`、`remote_trigger`
- **工具发现**：`tool_search` 让模型在工具多时按需检索；工具 schema 会进入系统提示/请求
- **输出治理**：`services/tool_outputs.py` 与超大输出的截断/落盘策略（`tool_artifacts`）
- **插件工具**：插件 `tools/` 目录下的 `BaseTool` 子类会被自动发现并注册（见 s09）

## 三、关键接口 / 符号

- `BaseTool`、`ToolRegistry.register()`、`ToolExecutionContext`、`ToolResult`
- `create_default_tool_registry(mcp_manager=None)`
- 命名约定：工具名**蛇形小写**（权限白黑名单依赖此约定）

## 四、动手实验

```powershell
uv run oh -p "用 grep 找出所有 TODO 并汇总"
uv run oh -p "/plugin list" ; uv run oh -p "/skills"
uv run oh --dry-run -p "帮我查一下内部工单系统"   # 观察命中的 skills/tools
```

任务：
1. 写一个 `BaseTool` 子类（例如"调用内部 HTTP API 查询订单状态"），要求：参数校验、超时、错误分级、结果结构化。
2. 把它注册进工具注册表并在真实会话中触发一次调用。
3. 为它写 pytest 单测（参照 `tests/test_tools/` 的结构）。
4. 列出"给模型用的描述"与"给工程师看的实现"之间的差异陷阱（描述不清 → 误调用）。

## 五、验收标准

- 能默写工具的四个核心类型名及其职责。
- 自定义工具能被模型在真实会话中调用成功，且失败时返回**可读错误**而非抛栈。
- 能解释为什么工具名要用 snake_case（结合 `permissions` 与 `swarm/_READ_ONLY_TOOLS`）。
- 能为工具设计"幂等性/副作用"标签，并说明企业审计需要哪些字段。

## 六、进阶思考

- 43 个内置工具全量暴露给模型，token 成本与误调用率如何权衡？`tool_search` 是否足以解决？
- 高危工具（`bash`）如何做"分级授权"？现有权限模型能否表达"只在某目录下允许"？
- 工具结果很大的场景（如全仓库 grep），落盘 + 摘要的阈值该怎么定？
- 若企业已有 OpenAPI 目录，你会批量生成工具还是走 MCP？
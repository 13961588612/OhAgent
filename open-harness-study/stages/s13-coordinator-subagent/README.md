# s13 · Coordinator 与 Subagent

> 前置：s12 ｜ 建议投入：10–12h ｜ 难度：★★★★

## 一、学习目标

1. 掌握 `AgentDefinition` 的全部字段，能声明式定义专家角色。
2. 理解子 Agent 的派生方式、上下文传递与结果回传。
3. 掌握协调者模式（Coordinator）的团队注册表与任务通知机制。

## 二、知识点清单

- **AgentDefinition**（`coordinator/agent_definitions.py`，约 46 KB）
  - 标识：`name` `description` `subagent_type`（默认 `general-purpose`）`source`（builtin/user/plugin）
  - 行为：`system_prompt` `initial_prompt` `critical_system_reminder`（每轮重复注入）
  - 能力：`tools`（`None`=全部，`['*']` 等价）/ `disallowed_tools`、`skills`、`mcp_servers` / `required_mcp_servers`
  - 模型与预算：`model`（`inherit` 表示继承父会话）、`effort`、`max_turns`
  - 治理：`permission_mode`、`permissions`（额外规则）、`hooks`（会话级钩子）
  - 记忆：`memory`（作用域）、`omit_claude_md`（跳过项目约定注入）
  - 运行形态：`background`（始终后台跑）、`isolation`（隔离模式）、`color`
  - 加载位置：内置 / 用户 / 插件；定义文件为 Markdown + frontmatter
- **派生机制**（`tools/agent_tool.py`）：主 Agent 通过工具调用派生子 Agent
- **协调者模式**（`coordinator/coordinator_mode.py`）
  - `TeamRegistry`：`create_team` / `delete_team` / `add_agent` / `send_message` / `list_teams`
  - `WorkerConfig`、`TaskNotification`（`format_task_notification` / `parse_task_notification` 走 XML 结构）
  - `is_coordinator_mode()`、`match_session_mode()`、`get_coordinator_tools()`、`get_coordinator_system_prompt()`
- **相关工具/命令**：`agent`、`send_message`、`team_create`、`team_delete`、`/agents`、`/subagents`
- **模型继承陷阱**：子 Agent 模型为 `inherit` 时通过 `OPENHARNESS_MODEL` 继承父会话，而非传字面量 `inherit`

## 三、关键接口 / 符号

- `AgentDefinition` 全字段（见上）
- `TeamRegistry`、`WorkerConfig`、`TaskNotification`
- `AgentTool`、`SendMessageTool`、`TeamCreateTool` / `TeamDeleteTool`

## 四、动手实验

```powershell
uv run oh -p "/agents"
uv run oh -p "/subagents"
```

任务：
1. 定义 3 个专家 Agent（如 `code-reviewer`、`sql-auditor`、`release-manager`），用 frontmatter 精确限制工具与权限（只读 + 指定 MCP + `permission_mode: plan`）。
2. 让主 Agent 派生它们完成一次真实任务，记录上下文如何传递、结果如何回传。
3. 给一个子 Agent 配置 `background: true`，观察与同步派生的差异。
4. 用 `critical_system_reminder` 强化一条硬约束（如"禁止直接改生产配置"），验证每轮都生效。
5. 设计"角色清单"：每类任务由谁负责、能否写、能访问哪些系统。

## 五、验收标准

- 能默写 AgentDefinition 的主要字段分组（标识/行为/能力/模型/治理/记忆/运行）。
- 自定义专家 Agent 能被成功派生且**遵守其工具与权限约束**（越权调用被拒）。
- 能解释 `model: inherit` 的传递机制与失败模式（非 Anthropic provider 下的历史坑）。
- 能说明协调者模式与"主 Agent 自己干"的取舍。

## 六、进阶思考

- 子 Agent 共享父会话的哪些状态？哪些必须隔离（工作目录、权限、记忆）？
- 多子 Agent 并发时，如何避免"上下文重复膨胀"（每个子 Agent 都带全量系统提示）？
- `critical_system_reminder` 每轮注入的成本与收益如何权衡？
- 若企业要求"每个专家角色都由配置中心下发"，AgentDefinition 的加载层需要怎样改造？
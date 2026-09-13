# 术语表（源码符号 ↔ 概念）

| 术语 | 含义 | 源码锚点 |
|------|------|----------|
| **Harness** | 智能体"外壳"：工具、权限、记忆、循环等模型之外的一切 | `src/openharness` |
| **Agent Loop** | "模型 → 工具 → 观察 → 再模型"的迭代循环 | `engine/query.py:run_query` |
| **QueryContext** | 单次循环的运行时上下文 | `engine/query.py:QueryContext` |
| **Tool** | 可被模型调用的能力单元 | `tools/base.py:BaseTool` |
| **ToolRegistry** | 工具注册表（按名解析、枚举 schema） | `tools/base.py:ToolRegistry` |
| **ToolResult** | 工具执行结果（内容、错误标记、元数据） | `tools/base.py:ToolResult` |
| **Permission Mode** | 权限档位：`default` / `plan` / `full_auto` | `permissions/modes.py` |
| **Path Rule / Denied Command** | 路径级与命令级黑白名单 | `settings.py:PermissionSettings` |
| **Hook** | 生命周期事件上的拦截/校验/通知逻辑 | `hooks/events.py`、`hooks/schemas.py` |
| **HookEvent** | 10 个事件：session_start/end、pre/post_compact、pre/post_tool_use、user_prompt_submit、notification、stop、subagent_stop | `hooks/events.py` |
| **Skill** | Markdown 描述的可复用工作流，可由模型或斜杠命令触发 | `skills/types.py:SkillDefinition` |
| **Plugin** | 打包 Skills/Tools/Commands/Agents/Hooks/MCP 的分发单元 | `plugins/schemas.py:PluginManifest` |
| **MCP** | Model Context Protocol：外部工具与资源的标准接入方式 | `mcp/types.py`、`mcp/client.py` |
| **Memory** | 跨会话持久知识（YAML frontmatter + 正文） | `memory/schema.py`、`memory/search.py` |
| **CLAUDE.md** | 项目级约定文件，自动发现并注入系统提示 | `prompts/claudemd.py` |
| **Session** | 可恢复的对话会话（存储、导出、标签、回滚） | `services/session_storage.py` |
| **Compact** | 上下文压缩：历史折叠成摘要以腾出窗口 | `services/compact/` |
| **Coordinator** | 协调者模式：主 Agent 调度子 Agent 团队 | `coordinator/coordinator_mode.py` |
| **Subagent** | 由主 Agent 派生、可带独立定义与权限的 Agent | `tools/agent_tool.py` |
| **AgentDefinition** | 子 Agent 的声明式定义（模型/工具/权限/记忆域） | `coordinator/agent_definitions.py` |
| **Swarm** | 多队友并行协作运行时（后端、邮箱、权限同步） | `swarm/` |
| **Teammate / Mailbox** | 队友身份与队友间消息邮箱 | `swarm/types.py`、`swarm/mailbox.py` |
| **Worktree** | 用 git worktree 为并行 Agent 做代码隔离 | `swarm/worktree.py` |
| **Task** | 后台任务（本地 shell / 本地 agent） | `tasks/manager.py` |
| **Channel** | 与 IM/邮件等外部入口对接的适配层 | `channels/impl/base.py` |
| **MessageBus** | 通道与引擎之间的解耦消息队列 | `channels/bus/queue.py` |
| **Gateway** | ohmo 的多通道/多会话路由服务 | `ohmo/gateway/service.py` |
| **ohmo** | 基于 OpenHarness 的个人智能体 App | `ohmo/cli.py`、`ohmo/workspace.py` |
| **Provider Profile** | 具体后端接入配置（协议、鉴权、模型、base_url） | `settings.py:ProviderProfile` |
| **Workflow** | 面向用户的接入方式分类 | `api/registry.py` |
| **Credential Slot** | profile 级独立凭据位（不再共用全局 key） | `settings.py:credential_slot` |
| **Sandbox** | 执行隔离（docker 后端 / SRT / 路径校验） | `sandbox/`、`utils/network_guard.py` |
| **Autopilot** | 面向仓库的自动化流水线（策略、运行、验证、发布） | `autopilot/service.py` |
| **Dry-run** | 不调用模型/工具，仅预览会话将如何运行 | `oh --dry-run` |
| **Transcript** | 会话记录（可导出/分享） | `/export`、`/share` |
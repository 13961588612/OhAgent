# OpenHarness 分层架构总览

> 基准：`HKUDS/OpenHarness` @ `9b2efd7`（v0.1.9）

## 一、一句话架构

**The model is the agent. The code is the harness.**
模型负责"想"（推理与决策）；Harness 负责"手、眼、记忆、安全边界"（执行、感知、记忆、约束）。

## 二、分层视图

```
┌──────────────────────────────────────────────────────────────┐
│ 交付层  channels/  ohmo/gateway/  ui/  frontend/terminal/     │
│         （IM 通道 · 网关路由 · TUI/React 前端）                │
├──────────────────────────────────────────────────────────────┤
│ 编排层  coordinator/ swarm/ tasks/ services/cron* autopilot/  │
│         （子智能体 · 团队协作 · 后台任务 · 定时 · 自动化）      │
├──────────────────────────────────────────────────────────────┤
│ 运行时  engine/（Agent Loop）  services/（记忆/压缩/会话）      │
├──────────────────────────────────────────────────────────────┤
│ 能力层  tools/ skills/ plugins/ mcp/ prompts/ memory/         │
├──────────────────────────────────────────────────────────────┤
│ 治理层  permissions/ hooks/ sandbox/ utils/network_guard.py   │
├──────────────────────────────────────────────────────────────┤
│ 接入层  api/（Provider 客户端） auth/（凭据） config/（配置）   │
├──────────────────────────────────────────────────────────────┤
│ LLM 后端  Anthropic 兼容 / OpenAI 兼容 / 订阅制 / 企业网关      │
└──────────────────────────────────────────────────────────────┘
```

## 三、模块清单

| 模块 | 职责 | 代表文件 |
|------|------|----------|
| `cli.py` | typer CLI 入口（`oh` / `openharness` / `openh`） | `cli.py`（约 96 KB，命令面最大） |
| `config/` | 配置解析与路径约定 | `settings.py`（Settings 全字段）、`paths.py`（~/.openharness） |
| `api/` | 各协议 Provider 客户端 | `client.py`（Anthropic）、`openai_client.py`、`codex_client.py`、`copilot_client.py`、`registry.py` |
| `auth/` | 认证与凭据存储 | `manager.py`、`storage.py`、`flows.py`、`external.py` |
| `engine/` | Agent Loop 与消息模型 | `query.py`、`query_engine.py`、`messages.py`、`stream_events.py`、`cost_tracker.py` |
| `tools/` | 内置工具与工具契约 | `base.py`、43 个内置工具文件 |
| `permissions/` | 权限模式与判定 | `modes.py`（default/plan/full_auto）、`checker.py` |
| `hooks/` | 生命周期钩子 | `events.py`（10 事件）、`schemas.py`（4 类）、`executor.py` |
| `skills/` | Markdown 技能 | `loader.py`、`bundled/content/*.md`、`types.py` |
| `plugins/` | 插件生态 | `loader.py`、`schemas.py`、`installer.py` |
| `mcp/` | MCP 客户端 | `client.py`、`types.py`（stdio/http/ws）、`config.py` |
| `memory/` | 记忆存储与检索 | `schema.py`、`search.py`、`relevance.py`、`manager.py`、`agent.py`、`team.py` |
| `prompts/` | 系统提示词与上下文注入 | `system_prompt.py`、`claudemd.py`、`context.py`、`environment.py` |
| `coordinator/` | 子智能体定义与协调模式 | `agent_definitions.py`、`coordinator_mode.py` |
| `swarm/` | 团队协作运行时 | `types.py`、`registry.py`、`mailbox.py`、`permission_sync.py`、`team_lifecycle.py`、`worktree.py` |
| `tasks/` | 后台任务管理 | `manager.py`、`types.py`、`local_agent_task.py`、`local_shell_task.py` |
| `services/` | 横切服务 | `compact/`、`token_estimation.py`、`session_storage.py`、`session_memory/`、`memory_extract/`、`autodream/`、`lsp/`、`cron*.py`、`tool_outputs.py` |
| `channels/` | IM 通道接入 | `adapter.py`、`bus/`、`impl/`（telegram/slack/discord/feishu/dingtalk/qq/email/matrix/whatsapp/mochat） |
| `ohmo/` | 个人智能体 App | `cli.py`、`workspace.py`、`prompts.py`、`memory.py`、`gateway/` |
| `ui/` | 终端交互 | `runtime.py`、`textual_app.py`、`react_launcher.py`、`backend_host.py`、`protocol.py`、`output.py` |
| `sandbox/` | 沙箱隔离 | `adapter.py`、`docker_backend.py`、`docker_image.py`、`path_validator.py`、`session.py` |
| `autopilot/` | 仓库自动化 | `service.py`（约 95 KB）、`types.py` |
| `themes/ keybindings/ vim/ voice/ output_styles/ personalization/` | 体验与个性化 | `builtin.py`、`resolver.py`、`voice_mode.py`、`extractor.py` |
| `utils/` | 通用工具 | `fs.py`、`file_lock.py`、`network_guard.py`、`shell.py` |

## 四、一次会话的数据流

```
用户输入（TUI / -p / IM 通道）
  → 配置与凭据解析        config/settings.py, auth/manager.py
  → 系统提示词组装        prompts/system_prompt.py + claudemd.py + context.py
  → 技能与工具装载        skills/loader.py, plugins/loader.py, mcp/client.py
  → 会话开始 Hook         hooks: session_start
  → Agent Loop           engine/query.py: run_query
        ├─ 调用模型       api/*（流式）
        ├─ 工具调用前     permissions/checker.py + hooks: pre_tool_use
        ├─ 执行工具       tools/*（bash/file/grep/mcp/task/agent...）
        ├─ 工具调用后     hooks: post_tool_use
        ├─ 上下文治理     services/compact/* + token_estimation.py
        └─ 多智能体编排   coordinator/ + swarm/ + tasks/
  → 记忆写入与提取        memory/*, session_memory, memory_extract
  → 会话结束 Hook         hooks: session_end（+ autodream 记忆固化）
  → 输出渲染              ui/output.py、ui/backend_host.py → React 前端
```

## 五、扩展点总表（企业二次开发入口）

| 想做什么 | 扩展方式 | 关键位置 |
|----------|----------|----------|
| 接入新模型/网关 | Provider profile / workflow | `api/registry.py`、`settings.py:ProviderProfile` |
| 新增工具 | `BaseTool` 子类 + 注册 | `tools/base.py`、`tools/__init__.py` |
| 固化团队规范 | Skill（`SKILL.md`） | `skills/loader.py`、`~/.openharness/skills` |
| 打包能力集 | Plugin（`plugin.json`） | `plugins/loader.py`、`plugins/schemas.py` |
| 接内部系统 | MCP Server | `mcp/types.py`、`mcp/client.py` |
| 强制合规 | Hooks | `hooks/schemas.py`、`hooks/executor.py` |
| 收紧权限 | 权限模式 + path rules + denied commands | `permissions/checker.py` |
| 隔离执行 | Sandbox | `sandbox/adapter.py` |
| 定义专家角色 | AgentDefinition | `coordinator/agent_definitions.py` |
| 接入 IM | Channel 实现 | `channels/impl/` |
| 无人值守自动化 | Autopilot / Cron | `autopilot/service.py`、`services/cron_scheduler.py` |

## 六、阅读顺序建议

1. 先读 `tools/base.py` —— 理解最小扩展单元；
2. 再读 `engine/query.py` —— 理解循环与治理插点；
3. 然后 `config/settings.py` —— 理解全部可配置面；
4. 最后按需深入巨型文件（`autopilot/service.py` 95 KB、`commands/registry.py` 126 KB、`cli.py` 96 KB）—— **不要顺序通读**。
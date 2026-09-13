# 源码文件级索引

> 基准：`.refs/OpenHarness`（`HKUDS/OpenHarness` @ `9b2efd7`）。用途：快速定位"某能力住在哪个文件"。
> 提示：本索引只写**职责**，不写实现细节——细节必须自己读源码。

## 顶层仓库

| 路径 | 说明 |
|------|------|
| `src/openharness/` | Harness 核心（实测 229 个 .py / 39,303 行，2026-09-12） |
| `ohmo/` | 个人智能体 App（含 `gateway/`） |
| `frontend/terminal/` | React 终端前端（`src/components/*`、`src/hooks/`、`src/theme/`） |
| `autopilot-dashboard/` | Vite + TS 看板 |
| `tests/` | 约 100 个测试模块，按模块镜像组织 |
| `scripts/` | 安装脚本与端到端验证脚本 |
| `docs/` | `SHOWCASE.md`、autopilot 静态站 |
| `.claude/skills/`、`.agents/skills/` | 仓库自带的技能示例（harness-eval、pr-merge） |
| `.github/workflows/` | `ci.yml`、autopilot 相关工作流 |
| `pyproject.toml` | hatchling 构建、依赖、入口点、pytest/ruff/mypy 配置 |
| `CHANGELOG.md` | 版本演进（读它可快速了解能力脉络） |

## 入口与配置

| 文件 | 职责 |
|------|------|
| `cli.py` | typer CLI 入口，所有 `oh` 命令（约 96 KB） |
| `__main__.py` | `python -m openharness` |
| `platforms.py` | 平台/能力探测 |
| `config/paths.py` | 配置/数据/日志/会话/任务/autopilot 路径解析 |
| `config/settings.py` | `Settings` 与全部配置模型、加载与保存（约 44 KB） |
| `config/schema.py` | 兼容性通道配置模型 |

## 接入层（Provider / Auth）

| 文件 | 职责 |
|------|------|
| `api/client.py` | Anthropic 客户端（含重试） |
| `api/openai_client.py` | OpenAI 兼容客户端（DashScope/OpenRouter/DeepSeek...） |
| `api/codex_client.py` | Codex 订阅（chatgpt.com Responses） |
| `api/copilot_client.py` | GitHub Copilot 客户端 |
| `api/copilot_auth.py` | Copilot OAuth device flow |
| `api/provider.py` | Provider/认证能力判定 |
| `api/registry.py` | workflow 与 provider preset 注册 |
| `api/usage.py` | 用量模型 |
| `api/errors.py` | API 错误类型 |
| `auth/manager.py` | 统一认证管理 |
| `auth/storage.py` | 凭据存储 |
| `auth/flows.py` | 各类认证流程 |
| `auth/external.py` | 复用外部 CLI 订阅凭据（Claude/Codex） |

## 运行时核心

| 文件 | 职责 |
|------|------|
| `engine/query.py` | 核心工具感知循环 `run_query`、`QueryContext`（约 40 KB） |
| `engine/query_engine.py` | 高层会话引擎 |
| `engine/messages.py` | 对话消息模型 |
| `engine/stream_events.py` | 流式事件类型 |
| `engine/cost_tracker.py` | 用量聚合与成本 |
| `ui/runtime.py` | `build_runtime`：装配 headless/TUI 运行时（约 32 KB） |

## 能力层

**Tools（`tools/`，43 个内置 + 契约）**

| 分类 | 文件 |
|------|------|
| 契约 | `base.py`（`BaseTool` `ToolRegistry` `ToolExecutionContext` `ToolResult`）、`__init__.py`（`create_default_tool_registry`） |
| 文件系统 | `file_read_tool.py` `file_write_tool.py` `file_edit_tool.py` `notebook_edit_tool.py` `glob_tool.py` `grep_tool.py` |
| 执行/环境 | `bash_tool.py` `sleep_tool.py` `enter_worktree_tool.py` `exit_worktree_tool.py` |
| 代码智能 | `lsp_tool.py` |
| 检索/网络 | `web_search_tool.py` `web_fetch_tool.py` `tool_search_tool.py` |
| 多模态 | `image_generation_tool.py` `image_to_text_tool.py` |
| 交互/协作 | `ask_user_question_tool.py` `send_message_tool.py` `brief_tool.py` |
| 规划/状态 | `enter_plan_mode_tool.py` `exit_plan_mode_tool.py` `todo_write_tool.py` `config_tool.py` |
| 技能 | `skill_tool.py` |
| MCP | `mcp_tool.py`（适配器） `list_mcp_resources_tool.py` `read_mcp_resource_tool.py` `mcp_auth_tool.py` |
| 任务/团队 | `task_*.py`（create/get/list/stop/output/update） `team_create_tool.py` `team_delete_tool.py` `agent_tool.py` |
| 定时/触发 | `cron_*.py`（create/list/delete/toggle） `remote_trigger_tool.py` |

**Skills（`skills/`）**

| 文件 | 职责 |
|------|------|
| `loader.py` | 多来源技能加载（bundled → user → compat → project → plugin） |
| `registry.py` / `types.py` | 技能注册表与 `SkillDefinition` |
| `_frontmatter.py` | YAML frontmatter 解析（bundled 与 user 共用） |
| `bundled/content/*.md` | 内置技能：`commit` `debug` `diagnose` `plan` `review` `simplify` `skill-creator` `test` |

**Plugins（`plugins/`）**

| 文件 | 职责 |
|------|------|
| `schemas.py` | `PluginManifest`（name/version/skills_dir/tools_dir/hooks_file/mcp_file/author/commands/agents/hooks） |
| `loader.py` | 插件发现与装载（user `~/.openharness/plugins`、project `.openharness/plugins`，兼容 `.claude-plugin/plugin.json`） |
| `types.py` | `LoadedPlugin`、`PluginCommandDefinition` |
| `installer.py` | 插件安装 |

**MCP（`mcp/`）**：`client.py`（连接管理）、`types.py`（stdio/http/ws 配置）、`config.py`（来自 settings 与插件的合并）

**Prompts（`prompts/`）**：`system_prompt.py`、`claudemd.py`（CLAUDE.md 发现注入）、`context.py`（上下文拼装）、`environment.py`（环境信息）

**Memory（`memory/`）**：`schema.py`、`search.py`（含中文分词）、`relevance.py`、`scan.py`、`usage.py`、`manager.py`、`memdir.py`、`agent.py`、`team.py`、`migrate.py`、`paths.py`

## 治理层

| 文件 | 职责 |
|------|------|
| `permissions/modes.py` | `PermissionMode`：default/plan/full_auto |
| `permissions/checker.py` | 权限判定（工具白黑名单、路径规则、命令黑名单） |
| `hooks/events.py` | 10 个 `HookEvent` |
| `hooks/schemas.py` | `CommandHookDefinition` `PromptHookDefinition` `HttpHookDefinition` `AgentHookDefinition`（含 `matcher` `priority` `block_on_failure` `timeout_seconds`） |
| `hooks/executor.py` | 钩子执行引擎（聚合结果、阻断判定） |
| `hooks/loader.py` / `hot_reload.py` | 从 settings 装载 / 热重载 |
| `hooks/types.py` | `HookResult` `AggregatedHookResult` |
| `sandbox/adapter.py` `docker_backend.py` `docker_image.py` `path_validator.py` `session.py` `Dockerfile` | 沙箱隔离实现 |
| `utils/network_guard.py` | 出网目标校验（SSRF 防护） |
| `utils/fs.py` `file_lock.py` `shell.py` `helpers.py` | 原子写、文件锁、shell、兼容助手 |

## 编排层

| 文件 | 职责 |
|------|------|
| `coordinator/agent_definitions.py` | `AgentDefinition` 加载与校验（约 46 KB） |
| `coordinator/coordinator_mode.py` | 协调者模式、`TeamRegistry`、`WorkerConfig`、任务通知 |
| `swarm/types.py` | 后端类型（subprocess/in_process/tmux/iterm2）、队友身份与消息 |
| `swarm/registry.py` | 后端注册表 |
| `swarm/mailbox.py` | 队友邮箱 |
| `swarm/permission_sync.py` | 跨进程权限同步（约 39 KB） |
| `swarm/team_lifecycle.py` | 团队生命周期 |
| `swarm/worktree.py` | git worktree 隔离 |
| `swarm/in_process.py` / `subprocess_backend.py` | 两种执行后端 |
| `swarm/spawn_utils.py` / `lockfile.py` | 派生与加锁 |
| `tasks/manager.py` | 后台任务管理（约 19 KB） |
| `tasks/types.py` `local_agent_task.py` `local_shell_task.py` `stop_task.py` | 任务模型与实现 |
| `services/cron.py` `cron_scheduler.py` | 定时任务与调度 |
| `autopilot/service.py` `types.py` | 仓库自动化（约 95 KB，最大单文件之一） |

## 服务与记忆治理

| 文件 | 职责 |
|------|------|
| `services/compact/__init__.py` | 上下文压缩（约 67 KB） |
| `services/token_estimation.py` | Token 估算 |
| `services/session_storage.py` / `session_backend.py` | 会话存储 |
| `services/session_memory/__init__.py` | 会话级记忆 |
| `services/memory_extract/__init__.py` | 记忆自动提取 |
| `services/autodream/*` | 记忆固化（`/dream`，含备份与锁） |
| `services/lsp/__init__.py` | LSP 集成 |
| `services/tool_outputs.py` | 工具输出处理 |
| `services/oauth/` | OAuth 占位 |

## 交付层

| 文件 | 职责 |
|------|------|
| `channels/adapter.py` | `ChannelBridge`：MessageBus ↔ QueryEngine |
| `channels/bus/queue.py` `events.py` | 消息总线 |
| `channels/impl/base.py` | 通道基类 |
| `channels/impl/{telegram,slack,discord,feishu,dingtalk,qq,email,matrix,whatsapp,mochat}.py` | 各通道实现（feishu 最大，约 54 KB） |
| `channels/impl/manager.py` | 通道管理器 |
| `ohmo/cli.py` `workspace.py` `runtime.py` `prompts.py` `memory.py` `session_storage.py` `group_registry.py` | ohmo 个人智能体 |
| `ohmo/gateway/{service,router,bridge,notify,config,models,provider_commands,group_tool}.py` | ohmo 网关 |
| `ui/textual_app.py` | 默认 Textual TUI |
| `ui/react_launcher.py` `backend_host.py` `protocol.py` | React 前端启动与 JSON-lines 协议 |
| `ui/output.py` `permission_dialog.py` `coordinator_drain.py` `app.py` `input.py` | 渲染与交互 |
| `themes/` `keybindings/` `vim/` `voice/` `output_styles/` `personalization/` | 主题、键位、Vim、语音、输出风格、个性化 |

## 测试与脚本

| 路径 | 说明 |
|------|------|
| `tests/test_swarm/*` | 协作运行时（12 个文件，覆盖最广） |
| `tests/test_tools/*` | 工具行为（bash/grep/mcp/任务/图像/web） |
| `tests/test_services/*` | 压缩、cron、会话、autodream |
| `tests/test_ui/*` | TUI、React 后端、权限模式 |
| `tests/test_channels/*` | 通道基础与安全（飞书/Telegram 安全用例） |
| `tests/fixtures/fake_mcp_server.py` | MCP 测试夹具 |
| `scripts/test_harness_features.py` `test_real_skills_plugins.py` `e2e_smoke.py` `test_docker_sandbox_e2e.py` `react_tui_e2e.py` | 端到端验证 |
| `scripts/install.sh` `install.ps1` `install_dev.sh` | 安装脚本 |
| `scripts/migrate_memory.py` | 记忆迁移 |

## 实测行数快照（s00 · 2026-09-12）

统计范围：`src/openharness/**/*.py`，共 **229 个文件 / 39,303 行**。

Top5 巨型文件（改动风险最高，二次开发前先读）：

| 行数 | 文件 | 成因判断 |
|---|---|---|
| 2590 | `commands/registry.py` | 集中注册全部斜杠命令元数据 |
| 2250 | `cli.py` | 聚合全部 Typer 选项与启动分支 |
| 2089 | `autopilot/service.py` | 仓库自动化流水线主体 |
| 1542 | `services/compact/__init__.py` | 上下文压缩策略与实现 |
| 1146 | `channels/impl/feishu.py` | 飞书通道（含长连接与卡片渲染） |

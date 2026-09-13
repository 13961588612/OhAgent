# CLI / 命令全索引

> 来源：`src/openharness/cli.py`、`src/openharness/commands/registry.py`、`ohmo/cli.py`

## 一、`oh`（= `openharness` = `openh`）

### 交互与批处理

| 用法 | 说明 |
|------|------|
| `oh` | 进入交互式 TUI（React/Ink 终端前端 + Textual 备用） |
| `oh -p "prompt"` | 非交互（headless）单轮执行 |
| `oh -p "..." --output-format json` | 结构化 JSON 输出（便于脚本/通道消费） |
| `oh -p "..." --output-format stream-json` | 流式 JSON 事件输出 |
| `oh --dry-run` | 安全预览：不调模型、不执行工具、不 spawn subagent、不连 MCP |
| `oh --dry-run -p "..." --output-format json` | 结构化预览结论 |

`--dry-run` 的 `readiness` 结论：`ready` / `warning` / `blocked`，并给出 `next actions`。

### 子命令组

| 命令 | 子命令 | 作用 |
|------|--------|------|
| `oh setup` | — | 统一配置入口：选 workflow → 认证 → 选 preset → 确认模型 → 激活 profile |
| `oh auth` | `login` `status` `logout` `switch` `copilot-login` `codex-login` `claude-login` `copilot-logout` | 认证与凭据管理 |
| `oh provider` | `list` `use` `add` `edit` `remove` | Provider profile 管理 |
| `oh config` | `show` `set` | 配置读写 |
| `oh mcp` | `list` `add` `remove` | MCP Server 管理 |
| `oh plugin` | `list` `install` `uninstall` | 插件管理 |
| `oh cron` | `start` `stop` `status` `list` `toggle` `history` `logs` | 本地定时任务 |
| `oh autopilot` | `status` `list` `add` `context` `journal` `scan` `run-next` `tick` `install-cron` `export-dashboard` | 仓库自动化流水线 |

`oh provider add` 关键参数：`--label --provider --api-format --auth-source --model --base-url --credential-slot --api-key`。

### 内置 workflow（`oh setup` 选项）

- `Anthropic-Compatible API`（Claude / Kimi / GLM / MiniMax / 企业 Anthropic 网关）
- `Claude Subscription`（复用 `~/.claude/.credentials.json`）
- `OpenAI-Compatible API`（OpenAI / OpenRouter / DashScope / DeepSeek / SiliconFlow / Gemini / Groq / Ollama / NVIDIA NIM / 企业 OpenAI 网关）
- `Codex Subscription`（复用 `~/.codex/auth.json`）
- `GitHub Copilot`（OAuth device flow）

## 二、`ohmo`（个人智能体 App）

| 命令 | 作用 |
|------|------|
| `ohmo init` | 初始化 `~/.ohmo` 工作区（`soul.md` `identity.md` `user.md` `BOOTSTRAP.md` `memory/` `gateway.json`） |
| `ohmo config` | 引导式配置 gateway 与 channel（Telegram / Slack / Discord / Feishu） |
| `ohmo` | 运行个人智能体 |
| `ohmo gateway run` / `status` / `restart` | 网关前台运行 / 状态 / 重启 |

## 三、斜杠命令（69 个，`oh --dry-run` 实测，2026-09-12；来自 `commands/registry.py`）

**会话与上下文**：`/help` `/exit`(`/quit`) `/clear` `/version` `/status` `/context` `/summary` `/compact` `/resume` `/session` `/export` `/share` `/copy` `/tag` `/rewind` `/files` `/init` `/continue` `/stop`

**成本与统计**：`/cost` `/usage` `/stats`

**记忆与钩子**：`/dream` `/memory` `/hooks`

**模型与运行参数**：`/provider` `/model` `/fast` `/effort` `/passes` `/turns` `/permissions` `/plan` `/privacy-settings` `/rate-limit-options`

**扩展与集成**：`/skills` `/plugin` `/reload-plugins` `/mcp` `/config` `/agents` `/subagents` `/tasks` `/bridge` `/channels`(通道相关)

**多智能体与自动化**：`/agents` `/subagents` `/tasks` `/autopilot` `/issue` `/pr_comments` `/ship` `/upgrade` `/release-notes`

**工程协作**：`/diff` `/branch` `/commit` `/doctor`

**体验**：`/theme` `/output-style` `/keybindings` `/vim` `/voice` `/onboarding` `/feedback` `/login` `/logout`

> 用户可调用的 Skills 会**自动注册为斜杠命令**（见 `_register_user_invocable_skill_commands`）。

## 四、关键环境变量

| 变量 | 作用 |
|------|------|
| `OPENHARNESS_CONFIG_DIR` | 覆盖配置目录（默认 `~/.openharness`） |
| `OPENHARNESS_DATA_DIR` | 覆盖数据目录（默认 `~/.openharness/data`） |
| `OPENHARNESS_LOGS_DIR` | 覆盖日志目录（默认 `~/.openharness/logs`） |
| `OPENHARNESS_MODEL` | 子进程/子 Agent 继承模型 |

## 五、文件与目录约定

```
~/.openharness/                    全局
├─ settings.json                   主配置
├─ skills/<skill>/SKILL.md         用户技能
├─ plugins/<plugin>/plugin.json    用户插件
└─ data/  logs/  sessions/  tasks/  feedback/  cron_jobs.json

<project>/.openharness/            项目级
├─ skills/  plugins/  issue.md  pr_comments.md
└─ autopilot/  registry.json  repo_journal.jsonl  active_repo_context.md
              autopilot_policy.yaml  verification_policy.yaml  release_policy.yaml  runs/

~/.ohmo/                           ohmo 个人智能体
├─ soul.md  identity.md  user.md  BOOTSTRAP.md  memory/  gateway.json
```
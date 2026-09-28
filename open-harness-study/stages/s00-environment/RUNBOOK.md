# s00 执行手册（RUNBOOK）

> 配套文档：`README.md`（本阶段的学习设计）｜ 产出落点：`../../notes/s00-环境与源码地图.md`
> 用法：从上到下逐步执行。每步固定包含【目的】【命令】【参数说明】【预期输出】【结果解读】【学到的知识】。
> 建议耗时：2–3 小时 ｜ 前置：无 ｜ 平台：Windows + PowerShell

---

## 0. 开始前的准备

### 0.1 打开工作目录

```powershell
cd D:\code\OhAgent\.refs\OpenHarness
```

【目的】所有实验都在上游快照内执行，保证每个结论都能指到"文件 + 符号"。
【参数说明】`.refs\OpenHarness` 是只读的上游源码快照，不要在这里写作业、不要 `git commit`。
【预期输出】PowerShell 提示符前的路径变成 `D:\code\OhAgent\.refs\OpenHarness`。用 `Get-Location` 确认。
【结果解读】学习工程（`open-harness-study\`）与源码快照（`.refs\OpenHarness`）刻意分离：快照可以随时按 commit 重建，你的产出独立于快照存在。
【学到的知识】为什么要有源码快照？因为"源码为准"的学习规则要求所有结论可复现；如果直接在会变动的分支上学习，半年后你的笔记就对不上了。

### 0.2 隔离实验环境（推荐）

```powershell
$env:OPENHARNESS_CONFIG_DIR = "D:\code\OhAgent\open-harness-study\capstone\.local\config"
$env:OPENHARNESS_DATA_DIR   = "D:\code\OhAgent\open-harness-study\capstone\.local\data"
$env:OPENHARNESS_LOGS_DIR   = "D:\code\OhAgent\open-harness-study\capstone\.local\logs"
```

【目的】把 oh 的配置、数据、日志重定向到学习工程目录，避免污染你日常的 `~/.openharness`。
【参数说明】

- `OPENHARNESS_CONFIG_DIR`：配置目录，默认 `C:\Users\13961\.openharness`，里面是 `settings.json`。
- `OPENHARNESS_DATA_DIR`：数据目录，放会话、任务、cron 记录。
- `OPENHARNESS_LOGS_DIR`：日志目录，`oh cron logs` 读的就是这里。
【预期输出】三条命令都不产生输出。用下面这行确认：

```powershell
echo "$env:OPENHARNESS_CONFIG_DIR`n$env:OPENHARNESS_DATA_DIR`n$env:OPENHARNESS_LOGS_DIR"
```

【结果解读】变量生效时，上一行会打印出你设置的三个路径。这三个变量在 `src/openharness/config/paths.py` 里被优先读取（第 19 / 41 / 58 行附近），优先级高于默认的 `~/.openharness`。
【注意事项】环境变量只对**当前 PowerShell 窗口**有效，关窗口即失效，需要重新设置。
【学到的知识】企业平台里这叫"环境隔离/租户隔离"：同一份二进制，靠外部注入的路径变量决定数据写到哪。理解这一点，后面做多租户时就不会想着改源码。

### 0.3 建立记录习惯

每执行完一步，把三样东西写进 `notes/s00-环境与源码地图.md`：**命令原文**、**原始输出**（可截断）、**你的意外之处**。第三样是笔记里最有价值的部分。

---



## 步骤 1 · 确认快照版本干净

```powershell
git rev-parse --short HEAD
git status --short
git log --oneline -3
```

【目的】锁定学习基线：确认你在哪个 commit 上，且没有任何本地改动。
【参数说明】

- `git rev-parse --short HEAD`：打印当前提交的短哈希。
- `git status --short`：列出工作区改动；无改动时**没有任何输出**。
- `git log --oneline -3`：看最近 3 次提交，了解版本节奏。
【预期输出】

```
9b2efd7
（git status --short 无输出）
9b2efd7 fix(config): preserve profile auth when overriding model
```

【结果解读】`9b2efd7` 就是本路线所有文档引用的基准 commit（对应 v0.1.9）。`git status` 无输出说明这是纯净快照。
【如果不一样怎么办】如果你看到其他哈希或本地改动，说明快照被污染过：用 `git stash` 或 `git checkout -- .` 恢复，再继续。
【学到的知识】任何"上游行为"结论都必须绑定 commit。commit 变了，结论可能失效——这就是为什么笔记里要写基准版本。

---



## 步骤 2 · 安装依赖并验证入口

```powershell
uv sync --extra dev
uv run oh --version
```

【目的】建立可运行的开发环境，并验证 CLI 入口可用。
【参数说明】

- `uv sync --extra dev`：按 `uv.lock` 创建/同步 `.venv`，并额外装 `dev` 依赖（pytest / ruff / mypy / pexpect）。
- `uv run oh --version`：在托管的虚拟环境里运行 `oh`，不污染全局 Python。
【预期输出】

```
（uv sync 末尾类似）Resolved 120 packages / Audited ... in ...
openharness 0.1.9
```

【结果解读】版本号来自 `pyproject.toml` 的 `version = "0.1.9"`。`uv run` 会自动使用 `.venv`，所以后面所有命令都写成 `uv run oh ...`。
【如果失败怎么办】网络受限时会卡在包下载。确认能访问 PyPI 后重试；已同步过则很快返回。
【学到的知识】`pyproject.toml` 里四个入口点：`oh` / `openharness` / `openh` 都指向 `openharness.cli:app`，`ohmo` 指向 `ohmo.cli:app`。前者是通用 Agent CLI，后者是独立的"个人智能体"App——这个切分是 s16 的主题之一。

---



## 步骤 3 · CLI 总览：读懂 `oh --help`

```powershell
uv run oh --help
```

【目的】建立 `oh` 的能力全貌，这是后续 22 个阶段反复回来的地方。
【参数说明】`--help` 只打印用法，不启动会话。
【预期输出】帮助按 **8 个分组**排版：


| 分组               | 关键项                                                                                                                          | 用途      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------- |
| Options          | `--version` `-v`、`--help`                                                                                                    | 基础信息    |
| Session          | `--continue` `-c`、`--resume` `-r`、`--name` `-n`                                                                              | 会话续接    |
| Model & Effort   | `--model` `-m`、`--effort`、`--verbose`、`--max-turns`                                                                          | 模型与推理档位 |
| Output           | `--print` `-p`、`--output-format`、`--dry-run`                                                                                 | 非交互输出   |
| Permissions      | `--permission-mode`、`--dangerously-skip-permissions`、`--allowed-tools`、`--disallowed-tools`                                  | 权限控制    |
| System & Context | `--system-prompt` `-s`、`--append-system-prompt`、`--settings`、`--base-url`、`--api-key` `-k`、`--bare`、`--api-format`、`--theme` | 上下文与接入  |
| Advanced         | `--debug` `-d`、`--mcp-config`                                                                                                | 调试与 MCP |
| Commands         | `setup` `mcp` `plugin` `auth` `provider` `config` `cron` `autopilot`                                                         | 8 个子命令  |


【结果解读】记住三条最重要的旗标：

- `-p/--print <prompt>`：headless 单轮执行，是所有自动化/通道调用的基础形态。
- `--dry-run`：只解析不执行，是排查配置问题的第一手段。
- `--permission-mode`：`default` / `plan` / `full_auto`，对应 s06 的权限体系。

【学到的知识】`--bare`（最小模式：跳过 hooks、plugins、MCP 与自动发现）的存在说明：这些能力是**运行时装配**进来的，而不是硬编码在核心。装配点见 `ui/runtime.py` 的 `build_runtime`。

### 3.1 分组用法手册（逐组示例）

> 以下命令均在 `D:\code\OhAgent\.refs\OpenHarness` 下用 `uv run` 执行。

【分组是怎么来的】这 8 个分组不是 Typer 自动生成的，而是 `cli.py` 里每个选项显式声明的 `rich_help_panel="..."`（如 `cli.py:2193` 的 `rich_help_panel="Session"`）聚合而成；最后的 `Commands` 分组来自 `cli.py:751-775` 的 `app.add_typer(...)`。所以**分组 = 代码里声明的能力域**，读分组等价于读代码结构。

| 分组 | 性质 | 首要命令 |
| --- | --- | --- |
| Options | 基础信息 | `oh --version` |
| Session | 会话续接 | `oh -c` / `oh -r` |
| Model & Effort | 模型与推理档位 | `oh -m sonnet --effort high` |
| Output | 非交互 / 自动化 | `oh -p "..." --output-format json` |
| Permissions | 权限边界 | `oh --permission-mode plan` |
| System & Context | 上下文与接入 | `--dry-run` 校验后 `-p` 实跑 |
| Advanced | 调试与 MCP | `oh -d` / `--mcp-config` |
| Commands | 8 个子命令 | `oh setup` |

**Options**

```powershell
uv run oh --version      # openharness 0.1.9
```

`--version` 是 eager 回调，命中后直接 `raise typer.Exit()`（`cli.py:745`），不进会话。用它确认当前跑的是快照里的 0.1.9，而不是全局安装的旧版。

**Session**

```powershell
uv run oh -n "s00 实验"        # 给本次会话起名
uv run oh -c                   # 继续当前目录最近一次会话
uv run oh -r                   # 无参数 → 打开最近 10 个会话的选择器
uv run oh -r <session_id>      # 按 ID 直接恢复
```

会话快照与 `cwd` 绑定。`-c` 在无快照时报 `No previous session found in this directory.` 并返回退出码 1；`-r` 无参数时走 `list_session_snapshots(cwd, limit=10)` 选择器。恢复时 `messages` 与 `tool_metadata` 会回灌给 `run_repl`（`cli.py:2485` 附近）——历史上下文是**还原**，不是重放。

**Model & Effort**

```powershell
uv run oh -m sonnet                # 别名（sonnet/opus）或完整 model id
uv run oh --effort high            # low / medium / high / xhigh|max
uv run oh --max-turns 5            # 交互模式是上限，--print 下强制生效
uv run oh --verbose
```

配置里已有默认 model 时，这里只是本次覆盖。配合 `-p` 做"同题不同档位"的对比实验最直观。

**Output**

```powershell
uv run oh --dry-run
uv run oh -p "Explain this repository" --output-format json
uv run oh -p "Explain this repository" --output-format stream-json
uv run oh --dry-run -p "Explain this repository"
```

三种形态的边界：

- `--dry-run`：解析 settings / auth / system prompt / skills / commands / tools / MCP 配置，但不调模型、不执行工具、不 spawn subagent、不连 MCP；输出 `ready/warning/blocked` 与 `next actions`。**未配置任何 key 时仍返回 0**。
- `-p`：真跑一轮。无凭据会报 `No API key configured.` 并返回退出码 1——正好用它确认失败路径。
- `--dry-run -p "..."`：把 prompt 也纳入解析（能看到准备发什么），但仍不调模型，是"我配置到底对不对"的第一诊断手段。

**Permissions**

```powershell
uv run oh --permission-mode plan          # default / plan / full_auto
uv run oh --allowed-tools "Read,Grep"     # 逗号或空格分隔
uv run oh --disallowed-tools "Bash"
uv run oh --dangerously-skip-permissions  # 仅限沙箱环境
```

推荐组合 `--permission-mode plan --dry-run`：先确认"工具集被允许成什么样"，再决定是否放开。日常目录不要用 `--dangerously-skip-permissions`。

**System & Context**

```powershell
uv run oh -s "你只输出简体中文"                          # 覆盖默认 system prompt
uv run oh --append-system-prompt "回复要带文件行号"        # 追加
uv run oh --settings '{"model":"sonnet"}'               # JSON 文件路径或内联 JSON
uv run oh --base-url https://gw.corp/anthropic -k <key> # 临时指向企业网关
uv run oh --api-format openai                           # anthropic(默认) / openai / copilot
uv run oh --bare                                        # 跳过 hooks/plugins/MCP/自动发现
uv run oh --theme dark                                  # default/dark/minimal/cyberpunk/solarized
```

两个关键点：

- `--bare` 的存在再次印证 hooks、plugins、MCP、自动发现都是**运行时装配**的（装配点 `ui/runtime.py` 的 `build_runtime`）。排障时先 `--bare` 排除扩展干扰。
- `--theme` 不在 Python 端渲染，它经 `OPENHARNESS_FRONTEND_CONFIG` 传给 React 前端（`App.tsx:53` 读 `config.theme`），运行中还会被 `state_snapshot` 的 `status.theme` 覆盖。

**Advanced**

```powershell
uv run oh -d                                # debug 日志
uv run oh --mcp-config ./mcp.local.json     # 临时加载 MCP（文件路径或 JSON 字符串）
```

`-d` 的日志走 stderr，而前端 spawn 后端时 stderr 是 `inherit`，所以调试图直接打在终端上、不污染协议流。

### 3.2 子命令实战示例（职责表见步骤 4）

**接入层**

```powershell
uv run oh auth status                      # 先看认证矩阵（步骤 6 展开）
uv run oh auth login                       # 无参走菜单
uv run oh auth login anthropic             # 或直接指定 provider
uv run oh auth switch                      # 切换当前 profile 的认证源
uv run oh auth codex-login                 # 绑定本地 Codex 订阅会话
uv run oh auth claude-login                # 绑定本地 Claude 订阅会话
uv run oh auth copilot-login               # GitHub Copilot device flow

uv run oh provider list
uv run oh provider add corp --label "公司网关" --provider anthropic `
    --api-format anthropic --auth-source api_key --model kimi-k2.5 `
    --base-url https://gw.corp/anthropic --allowed-model kimi-k2.5
uv run oh provider use corp
```

`provider add` 的必填是位置参数 `name` 加 5 个选项（`--label --provider --api-format --auth-source --model`）；可选里的 `--allowed-model`（模型白名单）与 `--base-url`（企业网关）是 s02 做企业接入的两个抓手，另有 `--context-window-tokens`、`--auto-compact-threshold-tokens` 控制自动压缩。

**配置层**

```powershell
uv run oh config show                  # 打印解析后的 settings JSON（步骤 8 展开）
uv run oh config set model sonnet      # 支持点号嵌套键
uv run oh config set theme dark
```

`config set` 写的是 `~/.openharness/settings.json`，而 `config show` 展示的是**合并解析后**的结果；两者对不上时，说明有更高优先级的来源（环境变量、`--settings`、profile）。

**扩展层**

```powershell
uv run oh mcp list
uv run oh mcp add fs '{"command":"npx","args":["-y","@modelcontextprotocol/server-filesystem","C:\\code"]}'
uv run oh mcp remove fs

uv run oh plugin list
uv run oh plugin install C:\path\to\plugin
uv run oh plugin uninstall <name>
```

`mcp add` 的第二个参数是整段 JSON 字符串，PowerShell 里用单引号包住、内部双引号原样传给 CLI。装完用 `oh mcp list` 验证，再用 `oh --dry-run` 看 MCP 是否被解析。

**自动化层**

```powershell
uv run oh cron status
uv run oh cron list
uv run oh cron toggle <name> false
uv run oh cron start

uv run oh autopilot status
uv run oh autopilot add "补齐 s00 笔记" --body "整理 CLI 分组" --cwd D:\code\OhAgent\.refs\OpenHarness
uv run oh autopilot scan
uv run oh autopilot tick
uv run oh autopilot run-next
uv run oh autopilot export-dashboard
```

`autopilot add` 的位置参数是 `[source] <title>`，`source` 默认 `manual_idea`，可选 `idea / ohmo / issue / pr / claude`；不带 `--cwd` 时默认当前仓库根。

### 3.3 一步不落的执行序列

```powershell
cd D:\code\OhAgent\.refs\OpenHarness
uv sync --extra dev                              # 1. 建环境
uv run oh --version                              # 2. 确认版本 0.1.9
uv run oh --help                                 # 3. 对照 3.1 的 8 个分组
uv run oh --dry-run                              # 4. 无 key 也返回 0，看 warning + next actions
uv run oh auth status                            # 5. 认证矩阵与当前激活项
uv run oh provider list                          # 6. profile 列表（可能为空）
uv run oh config show                            # 7. 解析后的 settings
uv run oh -p "回答 1+1" --output-format json     # 8. 无凭据 → No API key configured.，退出码 1
uv run oh --permission-mode plan --dry-run       # 9. 验证权限模式被解析
uv run oh                                        # 10. 进 TUI（React/Ink 前端）
```

| 步骤 | 预期 | 偏离时说明什么 |
| --- | --- | --- |
| 2 | `openharness 0.1.9` | 版本不符 → 用错了解释器，确认 `uv run` 前缀 |
| 4 | 退出码 0，含 `warning` 与 `next actions` | 解析阶段就崩 → settings/auth/skills 有结构问题 |
| 8 | `No API key configured.`，退出码 1 | 这是**预期**失败，验证的是失败路径 |
| 10 | 进入全屏 TUI | 卡在 `npm install` → 前端依赖未装齐 |

---

## 步骤 4 · 子命令巡览

```powershell
uv run oh setup --help
uv run oh auth --help
uv run oh provider --help
uv run oh config --help
uv run oh mcp --help
uv run oh plugin --help
uv run oh cron --help
uv run oh autopilot --help
```

【目的】把 8 个子命令的职责和子动作记成一张表。
【预期输出】汇总如下（均为实测）：


| 子命令         | 子动作                                                                                                  | 一句话职责                                   |
| ----------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `setup`     | （无子命令，可带 profile 参数）                                                                                 | 统一入口：选 workflow → 认证 → 选模型 → 激活 profile |
| `auth`      | `login` `status` `logout` `switch` `copilot-login` `codex-login` `claude-login` `copilot-logout`     | 认证与凭据                                   |
| `provider`  | `list` `use` `add` `edit` `remove`                                                                   | Provider profile 管理                     |
| `config`    | `show` `set`                                                                                         | 配置读写（`set` 写入 `settings.json`）          |
| `mcp`       | `list` `add` `remove`                                                                                | MCP Server 配置                           |
| `plugin`    | `list` `install` `uninstall`                                                                         | 插件管理                                    |
| `cron`      | `start` `stop` `status` `list` `toggle` `history` `logs`                                             | 本地定时任务                                  |
| `autopilot` | `status` `list` `add` `context` `journal` `scan` `run-next` `tick` `install-cron` `export-dashboard` | 仓库自动化流水线                                |


【结果解读】按能力层归类：接入（`auth`/`provider`）→ 配置（`config`）→ 扩展（`mcp`/`plugin`）→ 自动化（`cron`/`autopilot`）→ 交互入口（`setup`）。这正好是路线图第二部分到第四部分的主线。
【学到的知识】`oh provider add` 必填参数有 6 个（`--label --provider --api-format --auth-source --model`），可选参数里 `--allowed-model` 是**模型白名单**、`--base-url` 是**企业网关地址**——这两个是 s02 做企业接入的关键抓手。

---



## 步骤 5 · 认识第二个 App：`ohmo`

```powershell
uv run ohmo --help
```

【目的】分清 `oh` 与 `ohmo`：一个是通用 Agent CLI，一个是"个人智能体"App。
【预期输出】选项含 `-p/--print`、`--model`、`--profile`、`--workspace`、`--max-turns`、`--cwd`、`--resume`、`--continue`；子命令 7 个：`init` `config` `doctor` `memory` `soul` `user` `gateway`。
【结果解读】`ohmo` 的工作区默认是 `~/.ohmo`（`--workspace` 可覆盖），里面是 `soul.md`（人格）、`user.md`（用户画像）、`memory/`（记忆）、`gateway.json`（通道配置）。它复用 core，但有自己的身份与记忆体系。
【学到的知识】`ohmo gateway` 是把 Agent 接入 IM（飞书/Slack/Telegram/Discord）的入口，s16 会展开。现在只需记住：**core 提供引擎，App 提供人设与通道**。

---



## 步骤 6 · 看清认证矩阵：`oh auth status`

```powershell
uv run oh auth status
```

【目的】在不配置任何密钥的前提下，看清 oh 支持哪些认证来源、当前激活的是哪个。
【预期输出】两张表。

第一张是认证来源（12 行，实测）：

```
Auth sources:
Source                   State          Origin     Active
------------------------------------------------------------
Anthropic API key        missing        missing    <-- active
OpenAI API key           missing        missing
Codex subscription       missing        missing
Claude subscription      missing        missing
GitHub Copilot OAuth     missing        missing
DashScope API key        missing        missing
Bedrock credentials      missing        missing
Vertex credentials       missing        missing
Moonshot API key         missing        missing
Gemini API key           missing        missing
MiniMax API key          missing        missing
ModelScope API key       missing        missing
```

第二张是 provider profile 表（11 行，实测，节选）：

```
Provider profiles:
Profile              Provider           Auth source            State        Active
--------------------------------------------------------------------------------------------
claude-api           anthropic          anthropic_api_key      missing      <-- active
claude-subscription  anthropic_claude   claude_subscription    missing
openai-compatible    openai             openai_api_key         missing
codex                openai_codex       codex_subscription     missing
copilot              copilot            copilot_oauth          missing
moonshot             moonshot           moonshot_api_key       missing
gemini               gemini             gemini_api_key         missing
minimax              minimax            minimax_api_key        missing
nvidia               nvidia             nvidia_api_key         missing
qwen                 dashscope          dashscope_api_key      missing
modelscope           modelscope         modelscope_api_key     missing
```

【结果解读】三个层次要分清：

- **认证来源（auth source）**：密钥存在哪里（环境变量、本地订阅凭据文件、OAuth）。
- **provider**：运行时用哪个客户端（`anthropic` / `openai` / `anthropic_claude` / `openai_codex` / `copilot`）。
- **profile**：把上面两者 + 模型 + base_url 打包成的命名配置，是你在 `-m` / `--model` 之外最常切换的对象。
`<-- active` 标出当前生效项：默认是 `claude-api` profile + `anthropic_api_key`。
【学到的知识】企业场景里，"一个业务线一套 profile + 独立 credential slot"就是靠这套机制实现的。注意 `claude-subscription` / `codex` 这两类 profile：它们复用本地 CLI 的订阅凭据，不需要 API Key——这是 s02 的一个重要分支。

---



## 步骤 7 · 列出全部 profile：`oh provider list`

```powershell
uv run oh provider list
```

【目的】看每个 profile 的默认模型与 base_url，为 s02 的企业网关选型做铺垫。
【预期输出】11 行，每行两段（第一行是概要，带 `*` 的是激活项）：

```
* claude-api: Anthropic-Compatible API [missing auth]
    auth=anthropic_api_key model=default base_url=(default)
  claude-subscription: Claude Subscription [missing auth]
    auth=claude_subscription model=default base_url=(default)
  openai-compatible: OpenAI-Compatible API [missing auth]
    auth=openai_api_key model=gpt-5.4 base_url=(default)
  codex: Codex Subscription [missing auth]
    auth=codex_subscription model=gpt-5.4 base_url=(default)
  copilot: GitHub Copilot [missing auth]
    auth=copilot_oauth model=gpt-5.4 base_url=(default)
  moonshot: Moonshot (Kimi) [missing auth]
    auth=moonshot_api_key model=kimi-k2.5 base_url=https://api.moonshot.cn/v1
  gemini: Google Gemini [missing auth]
    auth=gemini_api_key model=gemini-2.5-flash base_url=https://generativelanguage.googleapis.com/v1beta/openai
  minimax: MiniMax [missing auth]
    auth=minimax_api_key model=MiniMax-M2.7 base_url=https://api.minimax.io/v1
  nvidia: NVIDIA NIM [missing auth]
    auth=nvidia_api_key model=openai/gpt-oss-120b base_url=https://integrate.api.nvidia.com/v1
  qwen: Qwen (DashScope) [missing auth]
    auth=dashscope_api_key model=qwen-plus base_url=https://dashscope.aliyuncs.com/compatible-mode/v1
  modelscope: ModelScope [missing auth]
    auth=modelscope_api_key model=deepseek-ai/DeepSeek-V4-Flash base_url=https://api-inference.modelscope.cn/v1
```

【结果解读】注意两点：

1. 内置 profile 里 `base_url=(default)` 的有 5 个（claude-api / claude-subscription / openai-compatible / codex / copilot），其余预置了各自的官方地址。
2. `base_url` 就是"换成企业网关"的插槽——不改代码，改这一项即可指向私有化部署。

【学到的知识】profile 是**声明式**的：provider（客户端实现）+ auth_source（凭据来源）+ model + base_url 四个维度正交组合。理解了这一点，s02 里做"多模型路由 + 凭据隔离"就是水到渠成。

---



## 步骤 8 · 看脱敏后的完整配置：`oh config show`

```powershell
uv run oh config show
```

【目的】这是你最接近"配置真相"的一条命令：它打印**解析后**的完整设置。
【预期输出】一段 JSON（实测节选）：

```json
{
  "api_key": "[REDACTED]",
  "model": "claude-sonnet-4-6",
  "max_tokens": "[REDACTED]",
  "base_url": null,
  "timeout": 30.0,
  "context_window_tokens": "[REDACTED]",
  "auto_compact_threshold_tokens": "[REDACTED]",
  "api_format": "anthropic",
  "provider": "anthropic",
  "active_profile": "claude-api",
  "profiles": {
    "claude-api": {
      "label": "Anthropic-Compatible API",
      "provider": "anthropic",
      "api_format": "anthropic",
      "auth_source": "anthropic_api_key",
      "default_model": "claude-sonnet-4-6",
      "base_url": null,
      "last_model": null,
      "credential_slot": "[REDACTED]",
      "allowed_models": [],
      "context_window_tokens": "[REDACTED]",
      "auto_compact_threshold_tokens": "[REDACTED]"
    }
  }
}
```

【结果解读】

- `[REDACTED]` 是**安全设计**：敏感字段（api_key、credential_slot、上下文阈值）在输出时被替换，所以这条命令可以安全地贴进工单。
- 顶层字段是"当前生效值"，`profiles` 是"全部可选 profile"。
- `allowed_models: []` 表示不限制；填上就变成模型白名单。
【学到的知识】**配置的"解析后视图"和"文件原文"是两回事**。原文在 `settings.json`，解析逻辑在 `src/openharness/config/settings.py`。排查配置问题时永远看解析后的结果，因为环境变量、默认值、profile 覆盖都在解析阶段合并。

---



## 步骤 9 · 核心实验：`oh --dry-run`（人类可读）

```powershell
uv run oh --dry-run
```

【目的】本阶段最重要的一条命令：在**完全不调用模型、不执行工具**的前提下，看清 oh 把你的环境解析成了什么样。
【参数说明】`--dry-run` 会解析 settings、auth、system prompt、skills、commands、tools、MCP 配置，然后打印报告并退出。
【预期输出】实测完整结构（分 8 段）：

```
OpenHarness Dry Run

Readiness
- level: warning
- Runtime client resolution failed. Interactive commands may still work, but model execution would fail.
- Authentication is missing, so live model execution would not start successfully.
- next actions:
  - If you expect a model call later, fix authentication or provider profile configuration first.
  - Run `oh auth login` or configure the active profile credentials before executing.

Execution
- cwd: D:\code\OhAgent\.refs\OpenHarness
- prompt: (none)
- entrypoint: interactive_session
- detail: OpenHarness would start and wait for user input. No model or tool call happens until you submit one.

Resolved Settings
- profile: claude-api (Anthropic-Compatible API)
- provider: anthropic
- api_format: anthropic
- model: claude-sonnet-4-6
- base_url: (default)
- permission_mode: default
- max_turns: 200
- effort: medium / passes=1

Validation
- auth: missing
- api client: error
- system prompt chars: 7977
- mcp: skipped in dry-run (configured only; external servers are not started)
- mcp config errors: 0

Discovery
- plugins: 0
- skills: 10
- slash commands: 69
- built-in tools: 39
- configured mcp servers: 0

Available Tools
- bash (required: command; optional: cwd, timeout_seconds)
- ask_user_question (required: question)
- read_file (required: path; optional: limit, offset)
- write_file (required: content, path; optional: create_directories)
- edit_file (required: new_str, old_str, path; optional: replace_all)
- notebook_edit (required: cell_index, new_source, path; optional: cell_type, create_if_missing, mode)
- lsp (required: operation; optional: character, file_path, line, query)
- mcp_auth (required: mode, server_name, value; optional: key)
- glob (required: pattern; optional: limit, root)
- grep (required: pattern; optional: case_sensitive, file_glob, limit, root)
- image_to_text (optional: image_data, image_path, max_tokens, media_type)
- image_generation (optional: background, image_paths, input_fidelity, mask_path)
- ... (+27 more)

Available Skills
- commit: Create clean, well-structured git commits.
- debug: Diagnose and fix bugs systematically.
- diagnose: Diagnose why an agent run failed, regressed, or produced unexpected output...
- harness-eval: This skill should be used when the user asks to "test the harness"...
- plan: Design an implementation plan before coding.
- pr-merge: This skill should be used when the user asks to "merge a PR"...
- review: Review code for bugs, security issues, and quality.
- simplify: Refactor code to be simpler and more maintainable.
- ... (+2 more)

System Prompt Preview
You are OpenHarness, an open-source AI coding assistant CLI. You are an interactive agent...
```

【结果解读】逐段理解：


| 段                     | 含义                                    | 你要回答的问题                                      |
| --------------------- | ------------------------------------- | -------------------------------------------- |
| Readiness             | 综合健康度：`ready` / `warning` / `blocked` | 为什么是 warning 而不是 blocked？                    |
| Execution             | 这次运行会怎么启动                             | `entrypoint: interactive_session` 代表什么？      |
| Resolved Settings     | 解析后的最终配置                              | 这些值来自文件、环境变量还是默认值？                           |
| Validation            | 各项自检结果                                | `api client: error` 与 `auth: missing` 是什么关系？ |
| Discovery             | 发现了多少扩展与能力                            | skills=10、commands=69、tools=39 从哪来？          |
| Available Tools       | 39 个内置工具的入参签名                         | 哪些工具会触发权限检查？                                 |
| Available Skills      | 已加载技能及其触发描述                           | 技能从哪些目录加载？                                   |
| System Prompt Preview | 首屏系统提示词                               | 7977 字符里都装了什么？                               |


【关键结论】无任何密钥时，`--dry-run` **不报错、不崩溃**，而是给出 `warning` + 可执行的 `next actions`。这正是 s00 验收标准要求的"可解释的结论"。
【附带发现】`mcp: skipped in dry-run (configured only; external servers are not started)` —— 这是 dry-run "解析但不执行"边界的直接证据。
【学到的知识】

1. `auth: missing` 是原因，`api client: error` 是后果。诊断问题时要顺着因果链走。
2. `entrypoint: interactive_session` 说明不带 `-p` 时，dry-run 预览的是"交互会话启动"；带上 `-p` 后 entrypoint 会变成 `model_prompt`（见下一步对比）。
3. 这条命令在无凭据、无网络的环境里也能跑——它是 CI 与交付前自检的天然工具。

---



## 步骤 10 · 结构化预览：`--dry-run -p ... --output-format json`

```powershell
uv run oh --dry-run -p "hello" --output-format json
```

【目的】把同一份预览变成机器可读的 JSON，方便做脚本化体检与配置漂移检测。
【参数说明】

- `-p "hello"`：给出一个 prompt，让 runtime 按 headless 路径解析。
- `--output-format json`：JSON 输出；另有 `text`（默认）与 `stream-json`（流式）。
【预期输出】实测节选：

```json
{
  "mode": "dry-run",
  "cwd": "D:\\code\\OhAgent\\.refs\\OpenHarness",
  "config_path": "C:\\Users\\13961\\.openharness\\settings.json",
  "prompt": "hello",
  "prompt_preview": "hello",
  "settings": {
    "active_profile": "claude-api",
    "profile_label": "Anthropic-Compatible API",
    "provider": "anthropic",
    "api_format": "anthropic",
    "model": "claude-sonnet-4-6",
    "base_url": "",
    "permission_mode": "default",
    "max_turns": 200,
    "effort": "medium",
    "passes": 1
  },
  "validation": {
    "auth_status": "missing",
    "api_client": {
      "status": "error",
      "detail": "runtime client could not be resolved with current auth/config"
    },
    "system_prompt_chars": 7977,
    "mcp_validation": "skipped in dry-run (configured only; external servers are not started)",
    "mcp_errors": 0
  },
  "entrypoint": {
    "kind": "model_prompt",
    "detail": "The first live step would be a model request. Exact tool calls and parameters are decided by the model at runtime."
  },
  "commands": [ ... ]
}
```

【结果解读】

- `entrypoint.kind` 从 `interactive_session` 变成 `model_prompt`——这就是步骤 9 里埋的那个对比答案：**加了** `-p`**，第一站就变成模型请求**。
- `config_path` 告诉你"你的配置到底从哪个文件读的"。如果你在 0.2 设了 `OPENHARNESS_CONFIG_DIR`，这里会显示你设置的新路径——这是验证隔离是否生效的最快方式。
- `commands` 数组是 69 个斜杠命令的机器可读定义。
【学到的知识】同一份配置，两种呈现：人类可读版用于人看，JSON 版用于程序看。企业平台做"发布前配置体检"就基于 JSON 版。

---



## 步骤 11 · 验证认证门禁：真跑一次 `-p`

```powershell
uv run oh -p "Explain this repository"
echo "退出码 = $LASTEXITCODE"
```

【目的】亲眼看到"没有凭据时，模型调用被拦住"，而不是只听说。
【预期输出】实测：

```
Error: No API key configured.
No credentials found for auth source 'anthropic_api_key'. Configure the matching provider or environment variable first.
Run `oh auth login` to set up authentication, or set the ANTHROPIC_API_KEY (or OPENAI_API_KEY) environment variable.
退出码 = 1
```

【结果解读】与 dry-run 的对比是本节精华：


| 形态             | 行为                 | 退出码 |
| -------------- | ------------------ | --- |
| `oh --dry-run` | 输出 warning 报告，正常结束 | 0   |
| `oh -p "..."`  | 明确报错，拒绝执行          | 1   |


**同一个"缺少凭据"的状态，预览时是信息，执行时是错误。** 这就是 dry-run 的设计意图。
【学到的知识】退出码是自动化集成的关键：`0` 可以进流水线，`1` 必须人工处理。做 CI 门禁时要区分"warning 但可继续"与"error 必须中断"。

---



## 步骤 12 · 模块行数统计（README 任务 1）

```powershell
$r='src\openharness'
Get-ChildItem $r -Recurse -File -Filter *.py |
  ForEach-Object { [pscustomobject]@{
    File  = $_.FullName.Replace((Resolve-Path $r).Path+'\','')
    Lines = (Get-Content $_.FullName | Measure-Object -Line).Lines } } |
  Sort-Object Lines -Descending | Select-Object -First 15 | Format-Table -AutoSize

$all = Get-ChildItem $r -Recurse -File -Filter *.py
"文件数: $($all.Count)"
"总行数: $(($all | ForEach-Object { (Get-Content $_.FullName | Measure-Object -Line).Lines } | Measure-Object -Sum).Sum)"
```

【目的】量化"内核有多大"，并找出风险最集中的巨型文件。
【参数说明】`-Recurse -File -Filter *.py` 递归只取 Python 文件；`Measure-Object -Line` 统计行数；`Replace(...)` 把绝对路径裁成相对路径。
【预期输出】实测：

```
File                             Lines
----                             -----
commands\registry.py              2590
cli.py                            2250
autopilot\service.py              2089
services\compact\__init__.py      1542
channels\impl\feishu.py           1146
config\settings.py                 944
engine\query.py                    941
swarm\permission_sync.py           919
ui\backend_host.py                 879
coordinator\agent_definitions.py   805
channels\impl\mochat.py            760
ui\runtime.py                      729
swarm\team_lifecycle.py            706
plugins\loader.py                  653
channels\impl\matrix.py            611

文件数: 229
总行数: 39303
```

【结果解读】Top5 的成因：


| 文件                             | 成因                          |
| ------------------------------ | --------------------------- |
| `commands/registry.py`         | 69 个斜杠命令的元数据集中注册在此          |
| `cli.py`                       | 全部 Typer 选项、8 个子命令、启动分支的聚合点 |
| `autopilot/service.py`         | 仓库自动化流水线（扫描→排队→执行→验证）主体     |
| `services/compact/__init__.py` | 上下文压缩策略（s12 主战场）            |
| `channels/impl/feishu.py`      | 飞书通道（长连接、卡片渲染、鉴权）           |


【学到的知识】行数不等于复杂度，但**改动冲突率**和行数强相关。这 5 个文件是二次开发时最可能和上游打架的地方——所以 s21 讲"如何在不动上游的前提下扩展"，针对的就是它们。

---



## 步骤 13 · 测试基线（README 实验第 5 条）

```powershell
uv run pytest -q -p no:cacheprovider
```

【目的】建立"当前环境能通过多少测试"的基线，未来每次升级都拿它对比。
【参数说明】`-q` 精简输出；`-p no:cacheprovider` 禁用缓存写入（沙箱/只读环境友好）。
【预期输出】实测：**收集阶段就中断**，报

```
ERROR tests/test_mcp/test_http_flow.py
ModuleNotFoundError: No module named 'mcp.server.fastmcp'.
This is mcp 2.x, where FastMCP was renamed to MCPServer ...
```

【结果解读】这不是你的错，也不是上游的 bug 现场——是**依赖范围问题**：`pyproject.toml` 只写 `mcp>=1.0.0`，没有上界，于是装到了 `mcp 2.2.0`，而 2.x 改了 API。
【继续跑完整基线】绕开这一项：

```powershell
uv run pytest -q -p no:cacheprovider --ignore=tests/test_mcp/test_http_flow.py
```

实测结果：

```
25 failed, 1122 passed, 11 skipped, 12 warnings in 275.64s (0:04:35)
```

【结果解读】基线数字 = **1122 通过 / 25 失败 / 11 跳过 / 约 4 分 35 秒**。失败集中在 Windows 与只读环境：swarm 的 posix 文件锁、任务管理器写文件、cron 时区断言、hooks 的 shell 转义等。
【怎么用这份基线】记进笔记，s20 建质量门禁时再决定：哪些用例应当按平台跳过、哪些必须修。**本阶段不修**。
【学到的知识】"测试全绿"不是基线的前提，"基线稳定可复现"才是。一个连基线都没有的项目，无法判断升级是否引入回归。

---



## 步骤 14 · 画端到端链路草图（README 任务 2）

【目的】先凭直觉画一张"用户输入 → 输出"的草图，后面每学一个阶段就回来修正一次。这张图会在 s04 定稿。
【做法】不看源码，先写出你认为的链路，标注每一层的**依据文件**（猜的也要标）。参考起点：

```
用户输入
  → openharness.cli:app            (cli.py，参数解析)
  → build_runtime                  (ui/runtime.py，装配设置/认证/工具/技能)
  → run_query                      (engine/query.py，工具感知循环)
  → api client                     (api/client.py 或 api/openai_client.py)
  → 模型返回 → 工具调用 → 权限检查 → hooks → 结果回注
  → 输出                           (ui/output.py / textual_app.py / backend_host.py)
```

【预期输出】一张你自己画的图（Mermaid / 手绘 / 文本均可），每个节点标注文件路径。
【结果解读】画不准是正常的——s00 只要求"先猜"。真正要记住的是**装配点在哪**：`ui/runtime.py` 的 `build_runtime`，它决定了哪些能力被接进这次会话。
【学到的知识】"装配"与"执行"分离是这个架构的核心特征：`--bare` 之所以能关掉 hooks/plugins/MCP，就是因为它们在装配阶段被选择性接入。

---



## 步骤 15 · 验收自测（合上笔记回答）

对照 `README.md` 的验收标准，逐条自问；答不出来就回到对应步骤重看：


| #   | 自测问题                                            | 对照步骤                           |
| --- | ----------------------------------------------- | ------------------------------ |
| 1   | 核心 / ohmo / 前端 / 测试四个顶层目录各自职责是什么？               | 步骤 5，`docs/04-source-index.md` |
| 2   | 没配任何 key 时，`oh --dry-run` 为什么能给出结论而不是报错？        | 步骤 9                           |
| 3   | `oh -p "..."` 和 `oh --dry-run -p "..."` 差在哪些环节？ | 步骤 9 / 10 / 11                 |
| 4   | 你的测试基线数字是多少？失败集中在哪些领域？                          | 步骤 13                          |
| 5   | 五个巨型文件分别是哪几个，为什么它们风险最高？                         | 步骤 12                          |
| 6   | 怎么让 oh 的配置不落到 `~/.openharness`？                 | 步骤 0.2                         |


全部能答，且笔记第 2、4、5 节是你自己写的，s00 才算过关。

---



## 附录 A · 命令速查表


| 命令                                                | 作用               | 是否只读         |
| ------------------------------------------------- | ---------------- | ------------ |
| `uv sync --extra dev`                             | 同步开发环境           | 是（写 `.venv`） |
| `uv run oh --version`                             | 版本               | 是            |
| `uv run oh --help`                                | 全部选项与子命令         | 是            |
| `uv run oh setup --help`                          | 统一配置入口说明         | 是            |
| `uv run oh auth status`                           | 认证来源与 profile 状态 | 是            |
| `uv run oh provider list`                         | 全部 profile 与默认模型 | 是            |
| `uv run oh config show`                           | 解析后的配置（脱敏）       | 是            |
| `uv run oh mcp list`                              | 已配置 MCP Server   | 是            |
| `uv run oh plugin list`                           | 已安装插件            | 是            |
| `uv run oh cron status` / `list`                  | 调度器与任务           | 是            |
| `uv run ohmo --help`                              | 个人智能体 App        | 是            |
| `uv run oh --dry-run`                             | 环境体检（人读）         | 是            |
| `uv run oh --dry-run -p "x" --output-format json` | 环境体检（机读）         | 是            |
| `uv run oh -p "..."`                              | 真实模型调用           | 否（需凭据）       |
| `uv run pytest -q -p no:cacheprovider`            | 测试基线             | 否（会写临时文件）    |




## 附录 B · 常见问题

**Q1：**`uv run pytest` **直接中断怎么办？**
加 `--ignore=tests/test_mcp/test_http_flow.py`。原因是 `mcp 2.x` 与上游代码不兼容；彻底解决可 `uv add "mcp<2"`（属 s10 议题，s00 不建议改动依赖）。

**Q2：**`oh --dry-run` **退出码是多少？**
实测 **0**。如果看到 1，通常是把输出接进了会截断的管道，或在真跑 `-p`。判定标准：无凭据时 dry-run 报告 `warning` 且退出码 0，`-p` 报错且退出码 1。

**Q3：中文输出乱码？**
PowerShell 读取文件时加 `-Encoding UTF8`；写入时用 `[System.IO.File]::WriteAllText(..., New-Object System.Text.UTF8Encoding($false))` 以避免 BOM。

**Q4：**`scripts/test_harness_features.py` **为什么跑不了？**
该脚本头部写明 *Uses kimi model for real API calls*，依赖 `ANTHROPIC_AUTH_TOKEN`，属于 s02 之后的内容。

**Q5：怎么确认配置没污染我的个人环境？**
运行 `uv run oh config show`，看输出里的 `config_path`（或 dry-run JSON 的 `config_path` 字段）。它应当指向你设置的 `OPENHARNESS_CONFIG_DIR`。

## 附录 C · 笔记落点对照


| 本手册内容                 | 写进笔记的哪一节     |
| --------------------- | ------------ |
| 步骤 1–5 的目录与入口点        | 第 1 节 源码地图   |
| 步骤 9–11 的解析与执行差异      | 第 2 节 机制理解   |
| 步骤 6–8、12–13 的命令与输出   | 第 3 节 实验与证据  |
| mcp 2.x、斜杠命令数量、25 个失败 | 第 4 节 疑问与待验证 |
| 依赖上界、基线缺失、平台矩阵        | 第 5 节 企业视角   |



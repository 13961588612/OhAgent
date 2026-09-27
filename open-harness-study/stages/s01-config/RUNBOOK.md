# s01 执行手册（RUNBOOK）

> 配套文档：`README.md`（本阶段的学习设计）｜ 产出落点：`../../notes/s01-配置系统与目录约定.md`
> 用法：从上到下逐步执行。每步固定包含【目的】【命令】【参数说明】【预期输出】【结果解读】【学到的知识】。
> 建议耗时：2–3 小时 ｜ 前置：s00 ｜ 平台：Windows + PowerShell
> 基准：`HKUDS/OpenHarness` @ `9b2efd7`（v0.1.9）｜ 实测日期：2026-09-16

---

## 0. 开始前的准备

### 0.1 打开工作目录

```powershell
cd C:\code\OhAgent\.refs\OpenHarness
```

【目的】本阶段所有命令都在上游快照内执行，保证"默认值""文件行号"都能对上源码。
【参数说明】`.refs\OpenHarness` 是只读快照，不要在这里写作业、不要 `git commit`。
【预期输出】提示符前路径变成 `C:\code\OhAgent\.refs\OpenHarness`。
【结果解读】s01 的主题是"配置"，而配置的**默认值全部来自源码**；快照一旦被改动，"默认值"就不再是上游的默认值。
【学到的知识】配置系统的第一性原理：任何配置项都要有一个"权威默认值"的出处。源码就是这里的权威。

### 0.2 隔离实验环境（本阶段必须做）

```powershell
$env:OPENHARNESS_CONFIG_DIR = "C:\code\OhAgent\open-harness-study\capstone\.local\config"
$env:OPENHARNESS_DATA_DIR   = "C:\code\OhAgent\open-harness-study\capstone\.local\data"
$env:OPENHARNESS_LOGS_DIR   = "C:\code\OhAgent\open-harness-study\capstone\.local\logs"
```

【目的】s00 做隔离是"推荐"，s01 是"必须"：本阶段会**真的写** `settings.json`，不隔离就会改到你日常的 `~/.openharness`。
【参数说明】三个变量的解析顺序见 `config/paths.py`（`OPENHARNESS_CONFIG_DIR` 在 22 行、`OPENHARNESS_DATA_DIR` 在 44 行、`OPENHARNESS_LOGS_DIR` 在 61 行），环境变量优先级永远高于 `~/.openharness`。
【预期输出】三条命令都没有输出。用下面这行确认：

```powershell
echo "$env:OPENHARNESS_CONFIG_DIR`n$env:OPENHARNESS_DATA_DIR`n$env:OPENHARNESS_LOGS_DIR"
```

【结果解读】打印出三个路径即生效。**环境变量只对当前 PowerShell 窗口有效**，新开窗口要重设。
【学到的知识】"配置目录可注入"是企业多租户的第一块砖：同一份二进制，靠外部注入的路径决定读写范围。s01 后面所有"写配置"的实验，都是在这条注入链上做。

### 0.3 建立记录习惯

每执行完一步，把三样东西写进 `notes/s01-配置系统与目录约定.md`：**命令原文**、**原始输出**（可截断）、**你的意外之处**。第三样最有价值——本手册下面每个"⚠ 上游缺陷"都是这么被发现的。

---

## 步骤 1 · 快照干净性检查（先做，别跳）

```powershell
git rev-parse --short HEAD
git status --short
git log --oneline -3
```

【目的】确认默认值的出处没被污染。
【预期输出】`9b2efd7`；`git status --short` 无输出。
【结果解读】`9b2efd7` 是全部阶段文档引用的基准 commit。`git status` 出现 ` D`（删除）或 `??`（未跟踪）就是污染信号，**必须先恢复再学习**，否则你测到的"默认值"可能是被人改过的。
【真实事故记录（2026-09-16，准备本手册时踩到）】
快照里 `src/openharness/tasks/` 的 6 个文件被整体搬到了 `src/openharness/prompts/tasks/`，`git status` 显示：

```
 D src/openharness/tasks/__init__.py
 D src/openharness/tasks/manager.py
 ...（共 6 个）
?? src/openharness/prompts/tasks/
```

后果是 `oh` **所有**命令都在导入链上崩溃：

```
ModuleNotFoundError: No module named 'openharness.tasks'
```

链路：`cli.py:2008` → `commands/registry.py:18` → `autopilot/service.py:40` → `swarm/subprocess_backend.py:20` → `openharness.tasks.manager`。
处置：确认那份副本与 HEAD 逐字节相同后，删除重复目录并 `git restore -- src/openharness/tasks`，`git status` 恢复为空。
【如果不一样怎么办】`git restore -- <路径>` 还原被删文件；未跟踪的重复目录对照 HEAD 确认无独有内容后再删。
【学到的知识】三类问题要分清：**导入链崩溃**（源码被移动，s01 之外的问题）伪装成"配置读取失败"最像；**配置解析失败**（JSON/类型错）；**凭据缺失**（认证）。排障先分清层次，再动手。

---

## 步骤 2 · 路径体系：oh 到底把文件放在哪

```powershell
rg -n "^def " src/openharness/config/paths.py
```

【目的】先建立"哪些目录存在、谁派生谁"的地图，再谈配置内容。
【参数说明】`rg -n "^def "` 只列出函数定义与行号，是读一个模块最快的入口。
【预期输出】20 个函数，分三族：

| 族 | 函数（行号） | 返回 |
|---|---|---|
| 全局 | `get_config_dir()` 15、`get_config_file_path()` 32、`get_data_dir()` 37、`get_logs_dir()` 54 | `~/.openharness/` 及其子目录 |
| 数据派生 | `get_sessions_dir()` 71、`get_tasks_dir()` 78、`get_feedback_dir()` 85、`get_feedback_log_path()` 92、`get_cron_registry_path()` 97 | `get_data_dir()/...` |
| 项目级 | `get_project_config_dir(cwd)` 102、`get_project_issue_file()` 109、`get_project_pr_comments_file()` 114、`get_project_autopilot_dir()` 119 + 7 个 autopilot 派生（126/131/136/141/146/151/156） | `<cwd>/.openharness/...` |

【结果解读】注意三点：

1. **每个 getter 都会 `mkdir`**（`paths.py:28`、`50`、`67`、`105` …）。这意味着"读路径"本身就带副作用——`oh --dry-run` 也会在你的项目目录里造出 `.openharness/`（步骤 9 实测）。
2. `get_project_config_dir` 用的是 `Path(cwd).resolve() / ".openharness"`（102–107 行），与你设置的 `OPENHARNESS_CONFIG_DIR` **无关**：前者跟"当前目录"走，后者跟"环境变量"走。
3. autopilot 那一族（`registry.json`、`repo_journal.jsonl`、`*_policy.yaml`、`runs/`）全部挂在**项目级**目录下，说明它是"按仓库"而不是"按用户"隔离的状态。

【学到的知识】配置系统设计里，路径函数同时承担"定位"和"确保存在"两件事，代价是任何只读操作也可能写盘。企业化时如果要做只读巡检（如配置漂移检测），必须清楚哪些"读"会顺带建目录。

---

## 步骤 3 · 默认值全景：33 个顶层字段

```powershell
uv run python -c "from openharness.config.settings import Settings as S; [print(n, '|', f.annotation, '|', f.default if f.default_factory is None else '<factory>') for n, f in S.model_fields.items()]"
```

（中文环境下若引号转义麻烦，可把这段脚本存成 `fields.py` 再 `uv run python fields.py`，内容见附录 A。）
【目的】得到"不配置任何东西时，oh 认为世界是什么样"的完整底座。这是 s01 唯一必须背下来的东西。
【参数说明】`Settings.model_fields` 是 Pydantic v2 的字段元数据；`Settings` 定义在 `config/settings.py:566`。
【预期输出】33 个字段，按"接入 / 行为 / UI / 多模态"四块（实测量）：

| # | 字段 | 类型 | 默认值 | 块 |
|---|---|---|---|---|
| 1 | `api_key` | str | `""` | 接入 |
| 2 | `model` | str | `claude-sonnet-4-6` | 接入 |
| 3 | `max_tokens` | int | `16384` | 接入 |
| 4 | `base_url` | str \| None | `None` | 接入 |
| 5 | `timeout` | float | `30.0` | 接入 |
| 6 | `context_window_tokens` | int \| None | `None` | 接入 |
| 7 | `auto_compact_threshold_tokens` | int \| None | `None` | 接入 |
| 8 | `api_format` | str | `anthropic` | 接入 |
| 9 | `provider` | str | `""` | 接入 |
| 10 | `active_profile` | str | `claude-api` | 接入 |
| 11 | `profiles` | dict | 11 个内置 profile | 接入 |
| 12 | `max_turns` | int | `200` | 接入 |
| 13 | `system_prompt` | str \| None | `None` | 行为 |
| 14 | `permission` | `PermissionSettings` | `mode=default`（见下） | 行为 |
| 15 | `hooks` | dict | `{}` | 行为 |
| 16 | `memory` | `MemorySettings` | `enabled=True, max_files=5, max_entrypoint_lines=200, max_entrypoint_bytes=25000, session_memory_enabled=True` | 行为 |
| 17 | `sandbox` | `SandboxSettings` | `enabled=False, backend='srt', fail_if_unavailable=False` | 行为 |
| 18 | `web` | `WebSettings` | `proxy=None, resolution_mode='auto'` | 行为 |
| 19 | `enabled_plugins` | dict | `{}` | 扩展 |
| 20 | `allow_project_plugins` | bool | `False` | 扩展 |
| 21 | `allow_project_skills` | bool | `True` | 扩展 |
| 22 | `project_skill_dirs` | list | `['.openharness/skills', '.agents/skills', '.claude/skills']` | 扩展 |
| 23 | `mcp_servers` | dict | `{}` | 扩展 |
| 24 | `theme` | str | `default` | UI |
| 25 | `output_style` | str | `default` | UI |
| 26 | `vim_mode` | bool | `False` | UI |
| 27 | `voice_mode` | bool | `False` | UI |
| 28 | `fast_mode` | bool | `False` | UI |
| 29 | `effort` | str | `medium` | UI |
| 30 | `passes` | int | `1` | UI |
| 31 | `verbose` | bool | `False` | UI |
| 32 | `vision` | `VisionModelConfig` | `model='', api_key='', base_url=''` | 多模态 |
| 33 | `image_generation` | `ImageGenerationConfig` | `provider='auto', model='gpt-image-2', codex_model='gpt-5.4'` | 多模态 |

`permission` 子字段（`settings.py:50`）：`mode=default`、`allowed_tools=[]`、`denied_tools=[]`、`path_rules=[]`、`denied_commands=[]`。

【结果解读】README 说的"8 个分组"（权限/记忆/网络/沙箱/Provider/运行时/扩展/多模态）在此表里能一一对应，但**扁平结构只有 33 个字段**——分组是概念分组，不是 JSON 层级。真正有层级的只有 `permission`、`memory`、`sandbox`、`web`、`vision`、`image_generation`、`profiles`。
【学到的知识】默认值即"产品姿态"：`permission.mode=default`（不自动放权）、`sandbox.enabled=False`（默认不隔离）、`allow_project_plugins=False`（默认不信项目内的插件）——这三条最能反映上游对"安全 vs 便利"的取舍。注意后两条的方向是相反的：技能默认信任项目，插件默认不信任。

---

## 步骤 4 · 解析后视图 vs 文件原文

```powershell
uv run oh config show
```

【目的】看"最终生效值"，而不是"文件里写了什么"。
【参数说明】`oh config show` = `cli.py:2005-2011`：`load_settings()` → `_settings_json_for_display()`。它**只读不写**。
【预期输出】一份 257 行的 JSON（实测），顺序与步骤 3 的字段表一致，末尾包含 `permission` / `hooks` / `memory` / `sandbox` / `web` / `mcp_servers` / `vision` / `image_generation` 等嵌套对象。
【结果解读】三个必须区分的概念：
1. **文件原文** = `$env:OPENHARNESS_CONFIG_DIR\settings.json`（磁盘上真正存的东西）；
2. **解析后视图** = `oh config show`（合并了环境变量、profile 化、默认值、脱敏后的结果）；
3. **运行时视图** = `oh --dry-run` 里 "Resolved Settings" 段落（s00 步骤 9 已见）。

对不上时，**永远是解析后视图赢**。排查顺序：`config show` → 若无异常再 `--dry-run`。
【学到的知识】配置系统对外必须提供一个"合并后"的读取口，否则用户永远在猜哪个值生效。企业平台同理：`show` 类命令是刚需，不是锦上添花。
【⚠ 上游缺陷】`config show` 的输出**不是**精确的解析后视图，见步骤 5。

---

## 步骤 5 · 脱敏边界：`[REDACTED]` 打在哪

```powershell
uv run oh config show | Select-String "REDACTED"
uv run python -c "from openharness.config.settings import load_settings as L; s=L(); print('max_tokens =', s.max_tokens)"
```

【目的】搞清楚"看到 `[REDACTED]`"到底是敏感字段，还是规则误伤。
【参数说明】脱敏实现在 `commands/registry.py:283-303`，黑名单在 `266-278`：

```python
_SECRET_KEY_PARTS = (
    "api_key", "apikey", "auth_token", "access_token", "refresh_token",
    "token", "secret", "password", ..., "credential", "private_key",
)
```

【预期输出】实测被脱敏的字段：

| 字段 | 是否真敏感 | 原因 |
|---|---|---|
| `api_key` | 是 | 命中 `api_key` |
| `credential_slot` | 否 | 命中 `credential` |
| `max_tokens` | 否 | 命中裸词 `token` |
| `context_window_tokens` | 否 | 命中裸词 `token` |
| `auto_compact_threshold_tokens` | 否 | 命中裸词 `token` |

对照：`uv run python ...` 打印出 `max_tokens = 16384`（真实值）。

【结果解读】`_is_secret_key` 用的是**子串包含**：`any(part in normalized for part in _SECRET_KEY_PARTS)`。黑名单里放裸词 `token` 会让所有含 `token` 的键名被误伤。影响是"看不到"，不是"泄露"，所以不算安全问题；但它会让**基于 `config show` 的配置漂移检测产生假告警**。
【学到的知识】脱敏规则永远是"宁可误伤"的：漏掉一个真密钥是事故，误伤十个普通字段只是不便。要读真实值就换一条路（Python API），而不是把黑名单改窄——改窄就轮到真密钥泄露了。
【顺带一提】值以 `bearer ` / `basic ` 开头的字符串也会被整体打码（`registry.py:297-298`）。

---

## 步骤 6 · 写配置的正确姿势：`oh config set`

```powershell
uv run oh config set theme dark
uv run oh config set allow_project_skills false
uv run oh config set max_turns 42
uv run oh config set project_skill_dirs ".openharness/skills,.agents/skills"
```

【目的】掌握唯一的 CLI 写配置入口，并确认它写到哪。
【参数说明】`cli.py:2014-2035`。两个位置参数 `key`、`value`；`key` 支持**点号嵌套**（`permission.mode`、`memory.enabled`），解析见 `_config_resolve_target`（`cli.py:1975-1985`）；类型强转见 `_config_coerce_value`（`cli.py:1988-2002`）：

- bool → 接受 `1/true/yes/on` 与 `0/false/no/off`
- int / float → 直接转换
- list → **按逗号切分**并去空白
- 其他 → 原样当字符串

【预期输出】

```
Updated theme
Updated allow_project_skills
Updated max_turns
Updated project_skill_dirs
```

每条退出码 0。
【结果解读】写入目标是 `get_config_file_path()`，即 `$env:OPENHARNESS_CONFIG_DIR\settings.json`（未设置时才是 `~/.openharness/settings.json`）。写盘走 `save_settings()`（`settings.py:1086`）：先 `sync_active_profile_from_flat_fields().materialize_active_profile()`，再用**文件锁 + 原子写**（`settings.json.lock` → 临时文件 → 替换）。
【学到的知识】"点号键 + 类型强转"是配置 CLI 的通用范式；而"锁 + 原子写"说明上游把 `settings.json` 当成**可能被并发访问的状态文件**（后台 cron、TUI、CLI 可能同时在跑）。企业平台里这个文件就是"用户级状态"，不属于源码，必须外置。

---

## 步骤 7 · 写配置的失败路径（3 个实测）

```powershell
uv run oh config set nope 1                              # 未知键
uv run oh config set max_turns abc                       # 类型不符
uv run oh config set permission.mode plan                # 嵌套枚举 ⚠ 会崩
```

【预期输出】依次：

```
1) Unknown config key: nope          （stderr，exit 1）
2) Error: invalid literal for int() with base 10: 'abc'   （stderr，exit 1）
3) AttributeError: 'str' object has no attribute 'value'  （stderr，exit 1）
```

【结果解读】前两条是**设计内**的错误处理（`cli.py:2025-2032` 捕获 `KeyError` 与 `ValueError`）。
第 3 条是**上游缺陷**：`permission.mode` 的类型是 `PermissionMode`（枚举），不是 bool/int/float/list/str，`_config_coerce_value` 走到最后的 `return raw`，把 `"plan"` 这个**字符串**赋给了枚举字段；Pydantic v2 默认不校验赋值，于是内存里 `settings.permission.mode` 变成了 `str`；随后 `save_settings()` → `sync_active_profile_from_flat_fields()` 里 `self.permission.mode.value`（`settings.py:693`）访问 `.value` 直接崩。

**⚠ 这意味着 README 的动手实验第一条 `uv run oh config set permission.mode plan` 在 v0.1.9 上不可用。**
受影响面：所有"枚举/复杂类型"字段（如 `permission.mode`、`memory.*` 里的 `Literal` 字段）。
绕过方案（实测可行）：

```powershell
# 直接改文件：pydantic 在"读"的时候会做校验，所以手写合法值没问题
$cfg = "$env:OPENHARNESS_CONFIG_DIR\settings.json"
$raw = [System.IO.File]::ReadAllText($cfg)
[System.IO.File]::WriteAllText($cfg, ($raw -replace '"mode": "default"', '"mode": "plan"'),
    (New-Object System.Text.UTF8Encoding($false)))
uv run oh config show | Select-String '"mode"'     # → "mode": "plan"
uv run oh --dry-run | Select-String 'permission_mode'   # → - permission_mode: plan
```

注意第三条命令崩溃时**文件不会被写坏**（崩溃发生在 `atomic_write_text` 之前），所以可以放心复现。

【学到的知识】"写配置"和"读配置"的健壮性必须对齐：上游的读路径有 Pydantic 校验，写路径却只做了 5 种类型的强转，中间的空档就变成了崩溃点。**遇到这类 bug，正确做法是记录"缺陷 + 影响面 + 绕过方案"，而不是去改上游源码**（学到 s12/s21 再回头看它的修复成本）。

---

## 步骤 8 · 环境变量覆盖的优先级（3 条实测）

```powershell
$env:OPENHARNESS_MODEL = "zzz-env-test"
uv run oh config show | Select-String '"model"'      # → "model": "zzz-env-test"
Remove-Item Env:OPENHARNESS_MODEL

$env:ANTHROPIC_MODEL = "aaa-env-test"
uv run oh config show | Select-String '"model"'      # → "model": "deepseek-flash"（被忽略！）
Remove-Item Env:ANTHROPIC_MODEL

$env:OPENHARNESS_MAX_TURNS = "42"
uv run oh config show | Select-String '"max_turns"'  # → "max_turns": 42
Remove-Item Env:OPENHARNESS_MAX_TURNS
```

【目的】验证"环境变量 > 配置文件"这条规则，并看清**例外**。
【参数说明】实现是 `_apply_env_overrides()`（`settings.py:928-1040`）。关键写法看 945-960 行：

```python
openharness_model = os.environ.get("OPENHARNESS_MODEL")
if openharness_model:
    updates["model"] = ...              # OPENHARNESS_* 无条件覆盖
elif not profile_has_explicit_model:
    anthropic_model = os.environ.get("ANTHROPIC_MODEL")   # 原生变量只在 profile 未显式指定时生效
```

【结果解读】

| 变量 | 类别 | 行为 |
|---|---|---|
| `OPENHARNESS_MODEL` | 专有 | **无条件覆盖**（显式用户意图） |
| `OPENHARNESS_BASE_URL` / `OPENHARNESS_MAX_TOKENS` / `OPENHARNESS_TIMEOUT` / `OPENHARNESS_MAX_TURNS` / `OPENHARNESS_CONTEXT_WINDOW_TOKENS` / `OPENHARNESS_AUTO_COMPACT_THRESHOLD_TOKENS` / `OPENHARNESS_PROVIDER` / `OPENHARNESS_API_FORMAT` | 专有 | 同上 |
| `ANTHROPIC_MODEL` / `ANTHROPIC_BASE_URL` / `OPENAI_BASE_URL` | 原生 | **仅当当前 profile 没显式配置该字段时**才生效 |

第 2 条实测被忽略，是因为当前 profile（`deepseek`）的 `last_model` 已经是 `deepseek-flash`——这就是"profile 显式值挡住原生环境变量"的效果。
【学到的知识】这是"兼容性双轨制"的典型代价：为了让从 Anthropic 官方 SDK 迁移过来的用户能直接用 `ANTHROPIC_*`，又不想让它覆盖平台自己管的 profile，就产生了"专有变量无条件、原生变量有条件"的不对称。企业网关接入时（s02），要能分清这两类变量的边界。

---

## 步骤 9 · 项目级 `.openharness/` 与作用域

```powershell
$proj = "C:\code\OhAgent\open-harness-study\capstone\.local\s01-lab\project"
New-Item -ItemType Directory -Force -Path $proj | Out-Null
cd $proj
uv run --project C:\code\OhAgent\.refs\OpenHarness oh --dry-run
Get-ChildItem $proj -Force -Recurse | Select-Object FullName
```

【目的】看清全局配置目录与项目级目录的**作用域差异**。
【参数说明】`--project` 让 `uv` 用快照里的虚拟环境，但**工作目录保持在你 cd 进去的项目目录**——这正是观察项目级行为的关键。`--dry-run` 不调模型，可以随便跑。
【预期输出】dry-run 正常输出（`exit 0`），项目目录下多出：

```
...\s01-lab\project\.openharness
...\s01-lab\project\.openharness\autopilot
...\s01-lab\project\.openharness\plugins
```

【结果解读】

| 维度 | 全局 | 项目级 |
|---|---|---|
| 路径 | `$env:OPENHARNESS_CONFIG_DIR`（默认 `~/.openharness/`） | `<cwd>/.openharness/`（`paths.py:102`） |
| 决定因素 | 环境变量 | **当前工作目录** |
| 谁在用 | `settings.json`、凭据、会话、cron | `issue.md`、`pr_comments.md`、`autopilot/*`（`registry.py:1006`、`1901`） |
| 目录内容 | 跨项目共享 | 只属于这个仓库 |

注意：`.openharness/` 是被上游 `.gitignore` 忽略的（快照 `.gitignore` 里有 `.openharness/`），所以它不会污染 `git status`——但也意味着**它不会被提交、不会被别人看到**，排查问题时容易忘记它的存在。
【学到的知识】"用户级配置 vs 项目级配置"是配置系统最经典的两个作用域。上游把"跨项目的偏好"放全局，把"跟着仓库走的状态"放项目级。企业平台里，项目级目录通常正好对应"某个团队的仓库"，是天然的租户边界。

---

## 步骤 10 · 项目级技能与 `allow_project_skills`

```powershell
# 1) 在项目级目录放一个技能
$skillDir = "$proj\.openharness\skills\hello-world"
New-Item -ItemType Directory -Force -Path $skillDir | Out-Null
@"
---
name: hello-world
description: s01 实验用技能
---

# Hello World
"@ | Set-Content "$skillDir\SKILL.md" -Encoding UTF8

# 2) 默认开关下看技能清单
cd $proj
uv run --project C:\code\OhAgent\.refs\OpenHarness oh --dry-run

# 3) 关掉项目技能，再看一次
cd C:\code\OhAgent\.refs\OpenHarness
uv run oh config set allow_project_skills false
cd $proj
uv run --project C:\code\OhAgent\.refs\OpenHarness oh --dry-run
uv run oh config set allow_project_skills true     # 记得还原
```

【目的】把"配置开关 → 实际行为"这条链路走通一次，并理解技能发现的目录范围。
【参数说明】技能文件必须是 `<root>/<技能名>/SKILL.md`（`skills/loader.py:162`、`177`）。搜索目录来自 `project_skill_dirs`，默认三个：`.openharness/skills`、`.agents/skills`、`.claude/skills`。
【预期输出】在 dry-run 的 `Available Skills` 段：

- 开关为 `true`：`hello-world` 出现，技能总数 8 → 9；
- 开关为 `false`：`hello-world` 消失，回到 8。

【结果解读】两点容易忽略：
1. **技能数量跟 `cwd` 走**：在快照目录里跑是 10 个（s00 的基线数字），在空项目目录里跑只有 8 个。做"配置漂移检测"时，必须先固定 `cwd`，否则数字不可比。
2. `allow_project_skills` 默认 `True`，而 `allow_project_plugins` 默认 `False`——同样是"项目提供的东西"，技能被信任、插件不被信任。理由是插件能注册 hooks 与工具（执行面更大），技能只是提示词素材。
【学到的知识】"扩展的信任级别应该按它能做什么来分级"，而不是按它是什么类型来分级。这是设计扩展系统时最值得抄的一条。
【Windows 提示】`Available Skills` 里的中文描述在 PowerShell 里可能显示成乱码（本项目描述是 UTF-8，控制台按 GBK 解读）。这是终端编码问题，不是配置问题。

---

## 步骤 11 · 故意写坏 `settings.json`

> 用**实验目录**做，别拿真实配置试：`$lab = "$env:OPENHARNESS_CONFIG_DIR\..\s01-lab\config"`，并在每次测试前把 `$env:OPENHARNESS_CONFIG_DIR` 指过去。

```powershell
$lab = "C:\code\OhAgent\open-harness-study\capstone\.local\s01-lab\config"
New-Item -ItemType Directory -Force -Path $lab | Out-Null
$env:OPENHARNESS_CONFIG_DIR = $lab
$enc = New-Object System.Text.UTF8Encoding($false)

# 1) 非法 JSON
[System.IO.File]::WriteAllText("$lab\settings.json", '{"theme": "dark",}', $enc)
uv run oh config show

# 2) 未知字段
[System.IO.File]::WriteAllText("$lab\settings.json", '{"unknown_field": 1, "theme": "solarized"}', $enc)
uv run oh config show | Select-String '"theme"'

# 3) 类型不符
[System.IO.File]::WriteAllText("$lab\settings.json", '{"max_turns": "abc"}', $enc)
uv run oh config show
```

【预期输出】

| 用例 | 结果 | 退出码 |
|---|---|---|
| 非法 JSON（尾逗号） | `json.JSONDecodeError: Expecting property name enclosed in double quotes: line 1 column 18` | 1 |
| 未知字段 | **静默忽略**，`"theme": "solarized"` 正常生效 | 0 |
| 类型不符 | Pydantic `ValidationError`：`Input should be a valid integer...` | 1 |

【结果解读】
- 解析入口是 `load_settings()`（`settings.py:1047`）：`json.loads(...)` 没有 try/except → 非法 JSON 直接抛栈；`Settings.model_validate(raw)` → 类型错误抛 Pydantic 错。
- 未知字段被忽略，是因为 `Settings` 没有配置 `extra="forbid"`（对比 `config/schema.py` 里的 `_CompatModel` 反而显式 `extra="allow"`，那是通道适配器的兼容层）。
- 三种错误都是**启动即失败**（exit 1），不是"降级运行"。所以上游的设计是"配置错误要立刻暴露"，与 `--dry-run` 的"可诊断优先"配合使用。
【学到的知识】`extra` 策略是个真实权衡：宽松（忽略未知字段）对版本升级友好——旧版本能读新版本的配置文件；严格（拒绝）能防拼写错误。上游选了宽松，代价是 `them: dark` 这种拼写错误会被无声吞掉，用户永远等不到 theme 生效。

---

## 步骤 12 · 斜杠命令 `/config` 与子命令 `oh config` 的差异

```powershell
# 子命令：支持点号嵌套键
uv run oh config set memory.enabled false
# 斜杠命令（在 TUI 会话内输入）：只支持顶层键
/config show
/config set theme dark
/config set permission.mode plan    # → Unknown config key
```

【目的】同一个"改配置"的意图，在两个入口下的能力边界不同。
【参数说明】斜杠命令实现在 `commands/registry.py:1135-1152`：它用 `if key not in Settings.model_fields` 判断合法性——**只查顶层字段名**，所以 `permission.mode` 会被判为未知键；类型强转用的是另一套 `_coerce_setting_value`（`registry.py:306-327`，支持 `Literal` 校验，比 CLI 的那套更严）。
【预期输出】

| 入口 | `set theme` | `set permission.mode` | 未知键提示 |
|---|---|---|---|
| `oh config set`（CLI） | ✅ | ⚠ 崩溃（步骤 7） | `Unknown config key: x` |
| `/config set`（TUI） | ✅ | ❌ 仅顶层，报未知键 | `Unknown config key: x` |

【结果解读】两个入口最终都调 `save_settings()`，写的是同一个文件，但**校验逻辑是两份独立实现**（`cli.py:_config_coerce_value` vs `registry.py:_coerce_setting_value`）。这是典型的"双入口漂移"：同一个功能写两遍，行为必然分叉——分叉点就在枚举字段（一个崩、一个报错）和嵌套键（一个支持、一个不支持）。
【学到的知识】配置写入口应该只有**一个**权威实现，其余入口调用它。上游这里的两份实现，正好是"为什么企业平台要收敛写路径"的现成反例。

---

## 步骤 13 · 优先级总表（读源码确认）

```powershell
uv run python -c "import openharness.config.settings as m; print(m.__doc__)"
rg -n "def merge_cli_overrides|def load_settings|def _apply_env_overrides|def save_settings" src/openharness/config/settings.py
```

【目的】把散落在多处的覆盖规则收敛成一张表。
【预期输出】`settings.py` 模块文档字符串（第 1-7 行）自述的优先级：

```
1. CLI arguments
2. Environment variables (ANTHROPIC_API_KEY, OPENHARNESS_MODEL, etc.)
3. Config file (~/.openharness/settings.json)
4. Defaults
```

【结果解读】结合实测，展开成一张完整表（从高到低）：

| 优先级 | 来源 | 实现位置 | 本手册实测 |
|---|---|---|---|
| 1 | CLI 参数（`--model`、`--settings`、`--permission-mode` …） | `Settings.merge_cli_overrides()`（`settings.py:868`） | s00 步骤 3 的分组表 |
| 2 | 专有环境变量 `OPENHARNESS_*` | `_apply_env_overrides()`（`settings.py:928`） | 步骤 8 |
| 3 | 原生环境变量 `ANTHROPIC_*` / `OPENAI_*` | 同上，但**仅当 profile 未显式配置** | 步骤 8 的 E5b |
| 4 | 活动 profile 的字段（`profiles[name]`） | `resolve_profile()` / `materialize_active_profile()`（`settings.py:629`、`639`） | 步骤 4 的 `deepseek` |
| 5 | `settings.json` 的顶层字段 | `load_settings()`（`settings.py:1047`） | 步骤 6 |
| 6 | 内置默认值 + 11 个内置 profile | `Settings` 字段默认值、`default_provider_profiles()`（`settings.py:196`） | 步骤 3 |

另外两个"旁路来源"：

- `OPENHARNESS_PROFILE`：只换**活动 profile 名**（`settings.py:1064`、`632`），不改具体字段；
- `--settings '<json>'`（CLI）：内联 JSON 直接并入（s00 步骤 3 的 System & Context 组）。

【学到的知识】"优先级"和"作用域"是两件事：环境变量作用域最小（只影响当前进程），优先级却很高。设计配置系统时，把"临时覆盖"做成环境变量/CLI 参数，把"长期状态"做成文件，这条分工能少掉 80% 的配置事故。

---

## 步骤 14 · 完成任务 1：Settings 字段表 + 企业标注

把步骤 3 的 33 行表抄进笔记，并加一列"企业环境必须调整"，建议这样标：

| 字段 | 建议 | 理由 |
|---|---|---|
| `api_key` | 必须外置 | 明文落在 `settings.json` 里 = 凭据泄漏面（本实验的隔离目录里就有明文 key） |
| `base_url` / `provider` / `api_format` | 必须改 | 指向企业网关 |
| `active_profile` | 必须改 | 默认 `claude-api` 走公网 |
| `permission.mode` | 必须定 | `full_auto` 在生产是不可接受的默认 |
| `sandbox.enabled` | 建议开 | 默认 `False`，等于不隔离 |
| `allow_project_plugins` | 保持 `False` | 项目内插件能注册工具与 hooks |
| `allow_project_skills` | 评估 | 默认 `True`，仓库内容会进提示词 |
| `memory.*` / `auto_*` | 必须定 | 决定会话数据留存与自动压缩行为（s11/s12） |
| `mcp_servers` | 由平台下发 | 允许用户自配 = 允许任意进程启动（s10） |
| `image_generation` / `vision` | 评估 | 默认指向 OpenAI/Codex 模型 |

【预期输出】一张你自己标注过的表 + 一句结论：**这 33 个字段里，真正"跨用户共享的配置"只有一小部分，其余都是用户级状态**——这是企业化配置分层的起点。
【学到的知识】把"用户偏好"（theme、vim_mode）和"平台策略"（base_url、permission.mode、mcp_servers）混在同一个文件里，是上游单用户模型的自然结果，也是企业化时必须拆开的地方。

---

## 步骤 15 · 验收自测（合上笔记回答）

| # | 自测问题 | 对照步骤 |
|---|---|---|
| 1 | Settings 有哪 4 个块、33 个顶层字段里哪几个是嵌套对象？ | 步骤 3 |
| 2 | `OPENHARNESS_CONFIG_DIR` 与项目级 `.openharness/` 的作用域差在哪？ | 步骤 2 / 9 |
| 3 | 为什么 `oh config show` 里看不到 `max_tokens` 的真实值？ | 步骤 5 |
| 4 | `oh config set permission.mode plan` 会发生什么？怎么绕过？ | 步骤 7 |
| 5 | 为什么设了 `ANTHROPIC_MODEL` 却没生效？ | 步骤 8 |
| 6 | 技能数量为什么在快照目录是 10、在空项目目录是 8？ | 步骤 10 |
| 7 | 写错 `settings.json` 的三种错法，哪些会 exit 1？ | 步骤 11 |
| 8 | 项目级插件默认禁用，开关叫什么？理由是什么？ | 步骤 3 / 10 |

全部能答，且笔记里留下了**你自己实测的输出**，s01 才算过关。

---

## 附录 A · 命令速查表

| 命令 | 作用 | 是否只读 |
|---|---|---|
| `uv run oh config show` | 解析后配置（脱敏，见步骤 5） | 是 |
| `uv run oh config set <key> <value>` | 写一个配置项（支持点号键） | 否（写 `settings.json`） |
| `uv run oh config --help` | 子命令列表 | 是 |
| `uv run oh --dry-run` | 环境体检，含 `Resolved Settings` | 是（但会创建项目级 `.openharness/`） |
| `uv run python -c "from openharness.config.settings import load_settings as L; print(L().model_dump())"` | 读**真实值**（不脱敏） | 是 |
| `uv run python fields.py` | 列 33 个字段与默认值（脚本见下） | 是 |
| `rg -n "^def " src/openharness/config/paths.py` | 路径函数索引 | 是 |
| `git status --short` | 快照纯净性 | 是 |

`fields.py`：

```python
# 依据 src/openharness/config/settings.py 的 Settings 模型，列出顶层字段与默认值
from openharness.config.settings import Settings

for name, field in Settings.model_fields.items():
    if field.default_factory is not None:
        try:
            default = field.default_factory()
        except Exception as exc:
            default = f"<factory:{type(exc).__name__}>"
    else:
        default = field.default
    annotation = getattr(field.annotation, "__name__", str(field.annotation))
    rendered = repr(default)
    if len(rendered) > 48:
        rendered = rendered[:45] + "..."
    print(f"{name}\t{annotation}\t{rendered}")
```

## 附录 B · 常见问题

**Q1：`git` 报 `detected dubious ownership`？**
快照目录的属主与当前用户不一致（换用户/换机器后常见）。按提示执行 `git config --global --add safe.directory C:/code/OhAgent/.refs/OpenHarness`，或临时用 `git -c safe.directory='*' -C <路径> ...`。

**Q2：`oh config set permission.mode plan` 崩了，是环境坏了吗？**
不是。是上游缺陷（步骤 7），`AttributeError: 'str' object has no attribute 'value'`。文件不会被写坏，按步骤 7 的绕过方案操作。

**Q3：`config show` 里 `max_tokens` 显示 `[REDACTED]`，是不是我的 key 泄露了？**
相反，是脱敏规则**过宽**（黑名单含裸词 `token`），属于显示问题。要看真实值用附录 A 的 Python 一行命令。

**Q4：改了配置但 `config show` 没变？**
按步骤 13 的优先级表从上往下找：CLI 参数 → `OPENHARNESS_*` → `ANTHROPIC_*` → profile → 文件，前面任何一层都能盖住你刚写进文件的值。

**Q5：实验做完了，怎么确认没污染个人配置？**
`cd .refs\OpenHarness` 后跑 `uv run oh config show`，看 `active_profile`/`base_url` 是否是你实验目录里的值；或直接检查 `$env:OPENHARNESS_CONFIG_DIR\settings.json` 的修改时间。

**Q6：`Available Skills` 的中文乱码？**
终端按 GBK 解码 UTF-8 造成。读文件时加 `-Encoding UTF8`，写文件用 `[System.IO.File]::WriteAllText(..., New-Object System.Text.UTF8Encoding($false))` 避免 BOM。

**Q7：`permission` 之外的嵌套对象能直接用 `config set` 改吗？**
`memory.enabled`、`sandbox.enabled` 这类 bool 字段可以；枚举、`Literal`、以及整个嵌套对象（如 `memory` 本身）不建议——见步骤 7 的失败机理。

## 附录 C · 笔记落点对照

| 本手册内容 | 写进笔记的哪一节 |
|---|---|
| 步骤 2、9 的路径与作用域 | 第 1 节 源码地图 |
| 步骤 3–8、13 的字段与优先级 | 第 2 节 机制理解 |
| 步骤 4–7、10–12 的命令与实测输出 | 第 3 节 实验与证据 |
| 步骤 5 的脱敏误伤、步骤 7 的枚举崩溃、步骤 11 的三种坏配置、步骤 12 的双入口漂移 | 第 4 节 疑问与待验证 |
| 步骤 14 的字段表与"必须调整"标注 | 第 5 节 企业视角 |
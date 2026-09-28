# OpenHarness 深度研习 · 企业级智能体工程

> **上游基准**：`HKUDS/OpenHarness` · commit `9b2efd7` · v0.1.9 · MIT
> **核心源码**：`src/openharness` 约 **39,300 行 Python**，外加 `ohmo/`（个人智能体）、`frontend/terminal/`（React 终端）、`autopilot-dashboard/`（Vite 看板）、`tests/`（约 100 个测试模块）
> **本地只读源码快照**：`D:\code\OhAgent\.refs\OpenHarness`

本目录不是"教程摘抄"，而是**源码驱动的研习工程**：每一阶段都要求"读懂源码 → 跑通实验 → 产出企业化成果"。

---

## 一、终局能力（结业标准）

学完本路线，你应当能够独立完成以下 8 件事：

1. 从源码构建并运行 OpenHarness（`oh` / `ohmo`），掌握全部 CLI 与交互模式（含 headless、JSON 输出、dry-run）。
2. 把任意企业模型网关（Anthropic / OpenAI 兼容、订阅制、私有化）接入为 **provider profile**，并做多模型路由与凭据隔离。
3. 开发并治理 **Tools / Skills / Plugins / Hooks / MCP** 五类扩展，把它们打包成可复用、可审计的企业资产。
4. 设计 **权限、沙箱、审计** 三位一体的安全边界，满足企业内部合规与最小权限要求。
5. 构建 **记忆与会话** 体系（项目记忆、个人记忆、自动压缩、成本可控），支撑长周期任务。
6. 用 **Coordinator / Subagent / Swarm** 编排多智能体，实现任务分解、并行执行、worktree 隔离与权限同步。
7. 通过 **Channels + ohmo Gateway** 把 Agent 交付到飞书 / Slack / Telegram / 邮件等真实入口，并实现多租户路由。
8. 完成 **可观测性、测试、打包、部署、运维**，交付可上线的企业级 Agent 后端。

---

## 二、目录结构

```
open-harness-study/
├─ README.md                    ← 本文件：路线总览与进度表
├─ docs/                        ← 全局参考：架构、术语、CLI 索引、源码索引
│  ├─ 00-roadmap.md             学习路线与阶段依赖
│  ├─ 01-architecture.md        分层架构 + 模块清单 + 数据流 + 扩展点
│  ├─ 02-glossary.md            术语表（源码符号 ↔ 概念）
│  ├─ 03-cli-index.md           `oh` / `ohmo` / 斜杠命令全索引
│  ├─ 04-source-index.md        源码文件级索引（按模块）
│  └─ 05-environment-baseline.md 环境与测试基线（跨阶段复用）
├─ stages/                      ← 23 个阶段，每个阶段一份 README（目标/知识点/实验/验收）
├─ notes/                       ← 个人笔记区（模板见 notes/README.md）
├─ capstone/                    ← 毕业项目工作区
└─ .refs/OpenHarness（在上级目录）← 上游源码只读快照
```

---

## 三、学习路线（6 个部分 · 23 个阶段）

阶段数目**不固定**，按能力依赖递进设计：每个阶段都可独立验收，也构成下一阶段的前置。

### 第一部分 · 地基（必须先行）

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s00-environment` | 环境搭建与源码地图 | `pyproject.toml`、`cli.py`、`scripts/` | 可运行环境 + 源码导航图 |
| `s01-config` | 配置系统与目录约定 | `config/paths.py`、`config/settings.py` | 完整 settings 心智模型 |
| `s02-provider-auth` | Provider 与认证体系 | `api/*`、`auth/*` | 企业网关 profile |

### 第二部分 · 运行时核心

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s03-prompts-context` | Prompt 与上下文工程 | `prompts/*`、`personalization/*` | 可定制系统提示词方案 |
| `s04-agent-loop` | Agent Loop 引擎 | `engine/*`、`ui/runtime.py` | 循环/流式事件读通 |
| `s05-tools` | Tools 体系与自定义工具 | `tools/*` | 自研工具（可注册可测试） |
| `s06-permissions-security` | 权限与安全治理 | `permissions/*`、`utils/network_guard.py` | 企业权限策略 |
| `s07-hooks` | Hooks 事件与合规 | `hooks/*` | 审计/拦截钩子 |
| `s08-skills` | Skills 技能体系 | `skills/*` | 企业技能库 |
| `s09-plugins` | Plugins 插件体系 | `plugins/*` | 企业插件包 |
| `s10-mcp` | MCP 集成 | `mcp/*`、`tools/mcp_*` | 内部系统 MCP Server |

### 第三部分 · 记忆与上下文治理

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s11-memory-session` | Memory 与 Session | `memory/*`、`services/session_*`、`services/memory_extract/*`、`services/autodream/*` | 记忆体系设计 |
| `s12-compaction-cost` | 上下文压缩与成本控制 | `services/compact/*`、`services/token_estimation.py`、`engine/cost_tracker.py` | 成本治理方案 |

### 第四部分 · 多智能体与自动化

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s13-coordinator-subagent` | Coordinator 与 Subagent | `coordinator/*`、`tools/agent_tool.py` | 自定义 Agent 定义集 |
| `s14-swarm` | Swarm 协作运行时 | `swarm/*`、`tasks/*` | 并行团队编排 |
| `s15-tasks-cron-autopilot` | 后台任务 / Cron / Autopilot | `tasks/*`、`services/cron*`、`autopilot/*` | 自动化流水线 |

### 第五部分 · 交付与人机界面

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s16-channels-ohmo-gateway` | Channels 与 ohmo Gateway | `channels/*`、`ohmo/*` | IM 多通道接入 |
| `s17-tui-frontend` | TUI 与前端协议 | `ui/*`、`frontend/terminal/src/*`、`themes/*` | 定制终端体验 |
| `s18-sandbox` | 沙箱隔离 | `sandbox/*`、`utils/fs.py` | 隔离与路径策略 |

### 第六部分 · 工程化、运营与毕业项目

| 阶段 | 主题 | 核心源码 | 关键产出 |
|------|------|----------|----------|
| `s19-observability` | 可观测性与运营 | `engine/cost_tracker.py`、`api/usage.py`、日志与审计 | OTel/成本看板方案 |
| `s20-testing-ci` | 测试、质量与 CI | `tests/*`、`scripts/*`、`.github/workflows/*` | 质量门禁 |
| `s21-packaging-contributing` | 打包、发布与二次开发 | `pyproject.toml`、`scripts/install.*`、扩展点清单 | 内部发行版 |
| `s22-capstone` | **毕业项目：企业级智能体平台** | 全栈综合 | 可上线 MVP + 运维手册 |

---

## 四、每个阶段怎么用

每份 `stages/*/README.md` 固定包含 6 节：

1. **学习目标**：该阶段结束时应具备的能力（可验证）。
2. **知识点清单**：条目化知识点，并标注对应源码路径与符号。
3. **关键接口**：需要记住的类型/函数/事件名，形成"API 肌肉记忆"。
4. **动手实验**：可直接执行的命令或待实现的小任务。
5. **验收标准**：判定"过关"的客观条件（含自测问题）。
6. **进阶思考**：面向资深工程师的设计权衡问题。

> 强烈建议：理解代码前**先写笔记**（`notes/`），实验**先在源码快照里读实现**，再用自己的工程复现。

---

## 五、环境前置

```powershell
# 1) 进入上游源码快照，建立可运行环境
cd D:\code\OhAgent\.refs\OpenHarness
uv sync --extra dev          # 安装核心 + 开发依赖
uv run oh --help             # 验证 CLI
uv run pytest -q             # 建立测试基线（s20 深入）

# 2) 可选：全局安装（脚本方式）
#   scripts/install.ps1  /  scripts/install.sh
```

- 必需：`uv`、`git`（已具备）；`node`（React 终端前端 `s17` 需要，已具备）
- 模型接入：在 `s02` 完成 provider profile 配置后，才能跑通真实会话

---

## 六、学习规则（硬性）

- **源码为准**：任何结论都要能在 `.refs/OpenHarness` 中指出文件与符号。
- **可运行**：每阶段的实验必须真实执行并记录输出（贴到 `notes/`）。
- **产出导向**：每阶段至少产出 1 份可复用资产（工具/技能/插件/策略/文档/脚本）。
- **企业视角**：每阶段结尾回答一句——"这条能力放进公司平台，缺口在哪里？"
- **不复制粘贴**：实验代码要自己写，并在注释中写清依据的源码位置。

---

## 七、进度追踪

| 阶段 | 状态 | 完成日期 | 产出物 |
|------|------|----------|--------|
| s00 | ◐ | | `notes/s00-环境与源码地图.md` · `docs/05-environment-baseline.md` |
| s01 | ◐ | | `notes/s01-配置系统与目录约定.md` · `stages/s01-config/RUNBOOK.md` |
| s02 | ☐ | | |
| s03 | ☐ | | |
| s04 | ☐ | | |
| s05 | ☐ | | |
| s06 | ☐ | | |
| s07 | ☐ | | |
| s08 | ☐ | | |
| s09 | ☐ | | |
| s10 | ☐ | | |
| s11 | ☐ | | |
| s12 | ☐ | | |
| s13 | ☐ | | |
| s14 | ☐ | | |
| s15 | ☐ | | |
| s16 | ☐ | | |
| s17 | ☐ | | |
| s18 | ☐ | | |
| s19 | ☐ | | |
| s20 | ☐ | | |
| s21 | ☐ | | |
| s22 | ☐ | | |

> 图例：☐ 未开始 ｜ ◐ 进行中 ｜ ✅ 已完成

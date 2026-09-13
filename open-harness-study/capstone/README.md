# capstone · 企业级智能体平台工作区

本目录是 `stages/s22-capstone` 的**实现工作区**。完成阶段 s00–s21 后在这里交付可上线的 Agent 后端 MVP。

## 建议结构

```
capstone/
├─ README.md                  ← 本文件：目标、结构、运行方式
├─ profile/                   模型接入与凭据规范（不存明文密钥）
│  ├─ providers.md            profile 命名规范与多环境矩阵
│  └─ settings.example.json   脱敏后的参考配置
├─ policies/                  权限与合规
│  ├─ permission-baseline.md  权限基线（工具/路径/命令）
│  ├─ hooks.json              审计与拦截钩子
│  └─ audit-schema.md         审计字段定义
├─ skills/                    企业技能库（SKILL.md 布局）
├─ plugins/corp-toolkit/      企业插件（plugin.json + tools/ + skills/ + hooks.json + mcp.json）
├─ mcp-servers/               内部系统 MCP Server（源码 + 用法）
├─ agents/                    专家 AgentDefinition（Markdown + frontmatter）
├─ observability/             埋点脚本、指标计算、事故手册
├─ deploy/                    部署与运维（配置、启动脚本、升级与回滚）
└─ evidence/                  验收证据：命令输出、测试报告、成本样例（脱敏）
```

## 运行约定

- 实验一律在**独立配置目录**下进行，避免污染个人环境：
  ```powershell
  $env:OPENHARNESS_CONFIG_DIR = "$PWD\capstone\.local\config"
  $env:OPENHARNESS_DATA_DIR   = "$PWD\capstone\.local\data"
  uv run oh --dry-run
  ```
- 所有密钥通过环境变量注入，**禁止**提交到仓库（`.gitignore` 已覆盖 `.env`）。
- 每完成一个需求（R1–R8），在 `evidence/` 下追加一份证据文件，命名 `R<n>-<主题>.md`。

## 验收对应关系

| 需求 | 主要落点 | 对应阶段 |
|------|----------|----------|
| R1 模型网关 | `profile/` | s02 |
| R2 能力治理 | `skills/` `plugins/` `mcp-servers/` | s08 s09 s10 |
| R3 安全边界 | `policies/` | s06 s07 s18 |
| R4 记忆与成本 | `profile/settings.example.json` + `evidence/` | s11 s12 |
| R5 多智能体 | `agents/` | s13 s14 |
| R6 IM 交付 | `deploy/` + 通道配置 | s16 |
| R7 可观测 | `observability/` | s19 |
| R8 质量与发布 | `deploy/` + 测试门禁 | s20 s21 |

> 验收清单与答辩题见 `../stages/s22-capstone/README.md`。
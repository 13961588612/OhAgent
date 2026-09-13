# s21 · 打包、发布与二次开发

> 前置：s20 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握项目构建与入口点配置，能产出**企业内部分发版**。
2. 掌握安装脚本与开发环境搭建，能规范团队协作流程。
3. 能识别全部扩展点，判断"配置可解 / 需要插件 / 必须改源码"。

## 二、知识点清单

- **构建与打包**（`pyproject.toml`）
  - 构建后端：hatchling；包：`src/openharness` + `ohmo`
  - 入口点：`oh` / `openharness` / `openh` → `openharness.cli:app`；`ohmo` → `ohmo.cli:app`
  - wheel 强制包含前端静态资源（`frontend/terminal` → `openharness/_frontend`）
  - 依赖与可选依赖（`dev`）
- **安装脚本**（`scripts/`）：`install.sh`（支持 `--from-source`、`--with-channels`）、`install.ps1`、`install_dev.sh`、`sync_nanobot_channels.sh`
- **数据迁移**：`memory/migrate.py`、`scripts/migrate_memory.py`
- **版本演进**：`CHANGELOG.md`（Unreleased / 0.1.9 / 0.1.8 ...）、`RELEASE_NOTES_v*.md`、`/release-notes`、`/upgrade`
- **扩展点决策树**（基于 s01–s19 的全部结论）
  | 需求 | 首选 | 次选 | 最后手段 |
  |------|------|------|----------|
  | 接新模型 | provider profile | 新增 preset | 新 client |
  | 加能力 | 自定义 Tool | MCP Server | 改内核 |
  | 固化流程 | Skill | 插件打包 | 改提示词模块 |
  | 接内部系统 | MCP | Tool | — |
  | 强制合规 | Hook | 权限基线 | 改 checker |
  | 换 UI | 前端协议定制 | 新 channel | 新 UI 后端 |
  | 换编排 | AgentDefinition | 自研协调逻辑 | 改 coordinator |
- **贡献上游**（`CONTRIBUTING.md`、`.github/PULL_REQUEST_TEMPLATE.md`、issue 模板）：分支、测试、变更说明

## 三、关键接口 / 符号

- `[project.scripts]`、`[tool.hatch.build.targets.wheel]`
- `uv sync --extra dev`、`uv run oh`
- `/release-notes`、`/upgrade`、`scripts/migrate_memory.py`

## 四、动手实验

```powershell
uv sync --extra dev
uv build            # 产出 wheel/sdist
uv run pytest -q
```

任务：
1. 构建企业内部分发版：改包名/版本号，产出 wheel 并在干净虚拟环境安装验证。
2. 写一份团队开发规范（分支策略、提交规范、必跑检查、升级流程）。
3. 针对"公司要统一网关 + 内部 MCP + 技能库"，列出**配置清单 + 插件清单 + 需改源码的清单**各 3 条。
4. 模拟一次上游升级（`git fetch` 到新 commit），评估破坏性变更与适配工作量（可只做分析）。

## 五、验收标准

- 能独立构建并安装可用的分发版。
- 能画出"扩展点决策树"，并对 5 个真实需求给出选型理由。
- 能说清哪些改动会**破坏兼容**（工具名、配置字段、协议字段、清单格式）。
- 有一套团队可直接执行的开发与升级规范。

## 六、进阶思考

- Fork 上游长期维护 vs 跟随上游定期合并，哪种更适合企业？给出判据。
- 自研扩展与上游重名冲突（工具名/技能名/插件名）时怎么办？
- 若要为上游贡献功能，哪些改动最容易被接受（扩展点内）？哪些会被拒（内核侵入）？
- 企业内部分发的版本策略：跟随上游小版本极快迭代，如何控制稳定性？
# s11 · Memory 与 Session 记忆体系

> 前置：s10 ｜ 建议投入：10–12h ｜ 难度：★★★★

## 一、学习目标

1. 说清 OpenHarness 的**三层记忆**：项目约定（CLAUDE.md）、项目记忆（MEMORY.md）、会话记忆。
2. 掌握记忆的存储格式、检索算法与相关性排序。
3. 能设计"跨会话、可检索、可治理"的企业知识记忆方案。

## 二、知识点清单

- **记忆模型与存储**（`memory/`）
  - `schema.py`：记忆条目结构（YAML frontmatter：`name` `description` `type` + 正文）
  - `manager.py` `memdir.py` `paths.py`：记忆目录与文件管理
  - `scan.py`：扫描记忆文件并解析 frontmatter
  - `search.py`：检索（元数据权重更高；分词器支持**汉字**，适合中文团队）
  - `relevance.py`：相关性打分
  - `usage.py`：记忆使用统计（可用于淘汰低价值记忆）
  - `agent.py`：Agent 级记忆域；`team.py`：团队级记忆
  - `migrate.py`：记忆格式迁移（`scripts/migrate_memory.py`）
- **项目约定注入**（`prompts/claudemd.py`）：`CLAUDE.md` 自动发现 + 截断阈值（`MemorySettings.max_files / max_entrypoint_lines / max_entrypoint_bytes`）
- **会话与持久化**（`services/`）
  - `session_storage.py` / `session_backend.py`：会话落盘（`~/.openharness/data/sessions/`）
  - `session_memory/`：会话级记忆
  - `memory_extract/`：从对话中自动提取记忆（`auto_extract_enabled`、`auto_extract_max_records`）
  - `autodream/`：记忆固化/整理（`/dream`；`auto_dream_enabled` `auto_dream_min_hours` `auto_dream_min_sessions`；含备份与加锁）
- **会话操作**：`/resume` `/session` `/export` `/share` `/tag` `/rewind` `/summary` `/compact`
- **ohmo 隔离**：ohmo 的 `/memory` 指向 `~/.ohmo/memory`，**不注入**项目记忆——个人记忆与项目记忆分离
- **配置开关**：`MemorySettings.enabled`、`session_memory_enabled`

## 三、关键接口 / 符号

- 记忆条目：frontmatter `name` / `description` / `type`
- `load_settings().memory.*`（`max_files`、`auto_compact_threshold_tokens`、`auto_extract_*`、`auto_dream_*`）
- 命令：`/memory`、`/dream`、`/resume`、`/tag`、`/rewind`、`/export`

## 四、动手实验

```powershell
uv run oh -p "/memory"
uv run oh -p "/dream"
uv run oh -p "/tag before-refactor" ; uv run oh -p "/rewind"
uv run oh -p "/session" ; uv run oh -p "/export"
```

任务：
1. 手工创建 3 条不同类型（事实/规范/决策）的记忆条目，验证 `/memory` 能列出、且能被检索命中（用中文关键词验证分词）。
2. 打开 `auto_extract_enabled`，跑一段包含明确事实的对话，检查自动提取结果的质量并调参 `auto_extract_max_records`。
3. 打开 `auto_dream_enabled`，验证 `/dream` 的固化流程与备份文件。
4. 设计企业记忆治理：分类分级（公共/部门/项目/个人）、保留期、脱敏、失效与冲突处理。
5. 制造"记忆污染"（写入一条错误记忆），观察它对回答的影响并设计回滚方式。

## 五、验收标准

- 能区分三层记忆的作用域与存储位置，并说明各自的注入时机。
- 中文检索能命中正文内容（不只是元数据）。
- 能解释 `/rewind` 与"记忆回滚"不是一回事（前者改会话，后者改记忆）。
- 产出企业记忆治理规范（含分类、保留期、脱敏、淘汰指标）。

## 六、进阶思考

- 记忆检索是**本地文件 + 打分**，没有向量库。什么规模下会失效？企业上量后该换成什么？
- `auto_extract` 会写入模型判断的"事实"，如何防止把幻觉固化成长期记忆？
- 多租户场景下记忆隔离如何保证（目录隔离 vs 加密 vs 独立存储）？
- 记忆与 RAG 的关系：什么该进记忆，什么该进检索索引？
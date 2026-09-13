# s08 · Skills 技能体系

> 前置：s07 ｜ 建议投入：8–10h ｜ 难度：★★☆

## 一、学习目标

1. 掌握 `SKILL.md` 规范与 YAML frontmatter 的全部字段。
2. 理解技能的多来源加载与**覆盖优先级**，能为企业设计技能库分层。
3. 能把团队规范固化成可被模型和用户双重触发的技能。

## 二、知识点清单

- **技能模型**（`skills/types.py:SkillDefinition`）：`name` `description` `content` `source` `path` `base_dir` `command_name` `display_name` `aliases` `user_invocable` `disable_model_invocation` `model` `argument_hint`
- **加载来源与顺序**（`skills/loader.py:load_skill_registry`）
  1. 内置 bundled（`skills/bundled/content/*.md`）
  2. 用户级（`~/.openharness/skills`）
  3. 兼容目录（`~/.claude/skills`、`~/.agents/skills`）
  4. 项目级（默认 `.openharness/skills`、`.agents/skills`、`.claude/skills`，可用 `project_skill_dirs` 覆盖；需 `allow_project_skills=true`）
  5. 插件贡献的技能
  - 目录布局：`<root>/<skill-dir>/SKILL.md`
  - 项目级发现范围：从 cwd **向上到 git root**，由浅到深，深层可覆盖浅层
- **frontmatter 字段**：`name`、`description`、`user-invocable`、`disable-model-invocation`、`model`、`argument-hint`（解析见 `skills/_frontmatter.py`，使用 `yaml.safe_load`，支持 `>` `|` 块标量）
- **斜杠命令联动**：`user-invocable` 的技能在 `commands/registry.py:_register_user_invocable_skill_commands` 中注册为 `/命令`，支持参数与模型覆盖
- **内置技能**：`commit` `debug` `diagnose` `plan` `review` `simplify` `skill-creator` `test`
- **安全校验**：`_valid_project_skill_dirs` 拒绝绝对路径与 `..`（防止越权加载）
- **对比 Plugin**：技能是"提示词资产"，插件是"可执行资产的分发单元"

## 三、关键接口 / 符号

- `SkillDefinition`、`SkillRegistry`
- `load_skill_registry(cwd, extra_skill_dirs, extra_plugin_roots, settings)`
- `get_user_skills_dir()`、`discover_project_skill_dirs(cwd, project_skill_dirs)`

## 四、动手实验

```powershell
uv run oh -p "/skills"
uv run oh -p "/debug"          # 触发内置技能（若可调用）
uv run oh -p "/skill-creator"  # 让技能帮你写技能
```

任务：
1. 写一个企业技能：例如"按公司规范生成数据库变更脚本"，要求含前置检查、必填信息、失败回滚说明。
2. 用 frontmatter 控制：`user-invocable: true`、`argument-hint: "<table>"`、`model: <指定模型>`、`disable-model-invocation: false`。
3. 验证项目级技能覆盖用户级同名技能的行为。
4. 设计企业技能库分层：平台级（强制）/ 部门级 / 项目级，并说明冲突解决策略。
5. 写出技能质量的评审清单（描述清晰度、触发条件、边界、禁忌）。

## 五、验收标准

- 能默写 5 层加载来源与顺序，并解释"后加载覆盖先加载"的实际影响。
- 自研技能能被模型**自动命中**（不靠斜杠命令）——说明描述写得好。
- 能解释 `disable-model-invocation` 的使用场景（危险/仅人工触发）。
- 产出一份企业技能编写规范 + 至少 2 个可用技能。

## 六、进阶思考

- 技能正文是**全量注入**还是按需加载？大量技能时 token 成本如何控制（结合 `tool_search` 思路）？
- `description` 决定模型能否命中；如何做"技能命中率"的量化评估？
- 技能与插件都可提供技能，团队应如何划分边界（复用 vs 治理）？
- 上游兼容 `anthropics/skills` 布局，这对"技能可移植性"意味着什么商业价值？
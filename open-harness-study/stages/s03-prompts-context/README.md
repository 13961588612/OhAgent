# s03 · Prompt 与上下文工程

> 前置：s02 ｜ 建议投入：6–8h ｜ 难度：★★☆

## 一、学习目标

1. 说清系统提示词是如何被"组装"出来的（不是一段静态字符串）。
2. 掌握 `CLAUDE.md` 的发现与注入机制，能用它固化团队约定。
3. 理解输出风格与个性化规则如何影响模型行为，并能做企业级定制。

## 二、知识点清单

- **系统提示词组装**（`prompts/system_prompt.py`）：基础人格 + 环境信息 + 项目约定 + 技能索引 + 工具说明
- **CLAUDE.md 机制**（`prompts/claudemd.py`）：自动发现、注入；受限项（`max_files`、`max_entrypoint_lines`、`max_entrypoint_bytes` 来自 `MemorySettings`）；AgentDefinition 可用 `omit_claude_md` 跳过
- **上下文拼装**（`prompts/context.py`）：消息历史 + 系统上下文 + 运行时状态
- **环境注入**（`prompts/environment.py`）：cwd、平台、git 状态等
- **输出风格**（`output_styles/loader.py`）：可选风格模板，`/output-style`
- **个性化**（`personalization/extractor.py` `rules.py` `session_hook.py`）：从会话中抽取偏好并形成规则
- **可观察性**：`/context` 查看当前生效的系统提示词——这是排查"模型为什么不听话"的第一入口

## 三、关键接口 / 符号

- `build_system_prompt(...)`（`prompts/system_prompt.py`）
- CLAUDE.md 发现与截断策略（`MemorySettings.max_*`）
- `/context`、`/output-style`、`/init`（生成项目 OpenHarness 文件）

## 四、动手实验

```powershell
uv run oh -p "/context"                    # 观察系统提示词实际内容
uv run oh -p "/status"
```

任务：
1. 在测试项目里分别放置根级与子目录 `CLAUDE.md`，验证发现范围与注入顺序。
2. 把 `max_entrypoint_lines` 调小，观察长文件被截断后的效果与风险。
3. 设计一份企业版"项目约定模板"（技术栈、目录规范、提交规范、禁止事项）。
4. 对比 `/output-style` 不同风格下同一问题的回答差异。

## 五、验收标准

- 能画出系统提示词的组成块与来源文件。
- 能解释"为什么子 Agent 有时看不到项目约定"（`omit_claude_md`）。
- 能说出上下文预算（`context_window_tokens`）与 CLAUDE.md 截断参数的关系。
- 产出一份可复用的企业项目约定模板。

## 六、进阶思考

- 静态 `CLAUDE.md` + 动态技能索引，与"把全部规范塞进系统提示"相比，token 成本与指令遵循度如何权衡？
- 个性化抽取（`personalization/`）会不会与合规冲突？企业版该如何界定可抽取范围？
- 若要支持"多语言团队"，提示词层需要哪些改造（语言检测、术语表注入）？
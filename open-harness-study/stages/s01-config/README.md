# s01 · 配置系统与目录约定

> 前置：s00 ｜ 建议投入：6–8h ｜ 难度：★★☆

## 一、学习目标

1. 吃透 `~/.openharness/settings.json` 的**全部字段分组**，形成配置心智模型。
2. 理解全局 / 项目级 / CLI / 环境变量四层配置的优先级与合并规则。
3. 能回答"某个行为想改，应该改哪个配置项"。

## 二、知识点清单

- **路径体系**（`config/paths.py`）
  - 全局：`~/.openharness/`（可用 `OPENHARNESS_CONFIG_DIR` 覆盖）、`data/`、`logs/`
  - 派生：`data/sessions/`、`data/tasks/`、`data/feedback/`、`data/cron_jobs.json`
  - 项目级：`<cwd>/.openharness/`（含 `issue.md`、`pr_comments.md`）
  - autopilot 目录族：`registry.json`、`repo_journal.jsonl`、`active_repo_context.md`、`autopilot_policy.yaml`、`verification_policy.yaml`、`release_policy.yaml`、`runs/`
- **配置模型**（`config/settings.py`）
  - 权限：`PermissionSettings`（`mode`、`allowed_tools`、`denied_tools`、`path_rules`、`denied_commands`）、`PathRuleConfig(pattern, allow)`
  - 记忆：`MemorySettings`（`enabled`、`max_files`、`max_entrypoint_lines/bytes`、`context_window_tokens`、`auto_compact_threshold_tokens`、`auto_extract_*`、`session_memory_enabled`、`auto_dream_*`）
  - 网络：`WebSettings`（`allowed_domains`、`denied_domains`）
  - 沙箱：`SandboxSettings`（`backend`、`fail_if_unavailable`、`enabled_platforms`、`network`、`filesystem`、`docker`）
  - Provider：`ProviderProfile`（`label/provider/api_format/auth_source/default_model/base_url/credential_slot/allowed_models/context_window_tokens/...`）+ `providers` 字典 + `active_profile`
  - 运行时：`max_turns`、`system_prompt`、`fast_mode`、`effort`、`passes`、`verbose`、`theme`、`output_style`、`vim_mode`、`voice_mode`
  - 扩展：`hooks`（事件 → HookDefinition 列表）、`enabled_plugins`、`allow_project_plugins`、`allow_project_skills`、`project_skill_dirs`、`mcp_servers`
  - 多模态：`VisionModelConfig`、`ImageGenerationConfig`
- **读取与更新**：`load_settings()` 与各类更新函数；`oh config show/set`、`/config`

## 三、关键接口 / 符号

- `Settings`、`PermissionSettings`、`MemorySettings`、`SandboxSettings`、`WebSettings`、`ProviderProfile`、`HookDefinition`
- `load_settings()`、`get_config_file_path()`、`get_project_config_dir(cwd)`
- 环境变量：`OPENHARNESS_CONFIG_DIR` / `OPENHARNESS_DATA_DIR` / `OPENHARNESS_LOGS_DIR`

## 四、动手实验

> **逐步执行手册：[`RUNBOOK.md`](RUNBOOK.md)** —— 15 个步骤，每个命令都给出完整格式、参数说明、预期输出、结果解读与对应知识点，按它从上到下执行即可。

```powershell
uv run python -c "from openharness.config.settings import load_settings; print(load_settings().model_dump_json(indent=2))"
uv run oh config show
uv run oh config set permission.mode plan
uv run oh --dry-run            # 观察配置变化对结论的影响
```

任务：
1. 用一张表列出 Settings 的**全部顶层字段**及默认值，并标注"企业环境必须调整"的项。
2. 在项目级 `.openharness/` 下放一份技能，验证 `allow_project_skills=false` 时的行为差异。
3. 故意写坏 `settings.json`（如非法 JSON、未知字段），观察报错与容错行为。

实测修正（2026-09-16）：

- `uv run oh config set permission.mode plan`（命令速览里的第一条写操作）在 v0.1.9 上会崩：`AttributeError: 'str' object has no attribute 'value'`。原因是 `_config_coerce_value` 只支持 bool/int/float/list 强转，枚举字段被赋成字符串后，`save_settings()` 里访问 `.value` 失败；文件不会被写坏。绕过方式见 RUNBOOK 步骤 7。
- `uv run oh config show` 的脱敏规则包含裸词 `token` / `credential`，因此 `max_tokens`、`context_window_tokens`、`credential_slot` 会被一并打成 `[REDACTED]`；要看真实值改用 Python API（RUNBOOK 步骤 5）。
- 技能数量与 **cwd** 有关：在快照目录是 10 个技能，在空项目目录只有 8 个。做配置漂移比对前先固定 `cwd`（RUNBOOK 步骤 10）。
- 项目级 `.openharness/` 由 `get_project_config_dir()` 的 `mkdir` 副作用创建：在任意目录跑一次 `oh --dry-run`，就会生成 `.openharness/autopilot` 与 `.openharness/plugins`（RUNBOOK 步骤 9）。

## 五、验收标准

- 能凭记忆报出 Settings 的字段分组（权限/记忆/网络/沙箱/Provider/运行时/扩展/多模态）。
- 能解释 `OPENHARNESS_CONFIG_DIR` 与项目级 `.openharness/` 的作用域差异。
- 能说明"项目级插件默认禁用"的理由，并指出打开它的开关。

## 六、进阶思考

- 全局 `settings.json` 是**单用户**模型；企业要"多租户 + 集中下发"配置，应该在哪个层做覆盖（env / profile / 自研启动器）？
- `credential_slot` 解决了什么问题？如果企业用统一密钥管理服务（Vault/KMS），接入点在哪？
- 配置错误会导致 `blocked` 而非直接崩溃——这种"可诊断优先"的设计，对你的平台有何启发？
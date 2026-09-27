# 主题 · config 源码阅读路线（s01 配套）

> 用途：s01 的精读清单。基准 `HKUDS/OpenHarness` @ `9b2efd7`（v0.1.9）｜ 建立日期：2026-09-17
> 用法：按顺序读；每读完一段在 `[ ]` 打勾，并在第六节写一句话。**能复述才算读懂。**

---

## 一、读之前先明确三件事

1. 目标不是"读完"，而是能回答第三节的三个问题。
2. 不要通读 `settings.py`（1104 行）——按下方区间跳读，有效阅读量约 800 行。
3. 每一段都要能指出"文件 + 符号 + 一句话职责"，沿用 s00 的源码地图规则。

## 二、阅读清单

### 1. `config/paths.py`（160 行，建议通读）
- [ ] 读完
- 重点：三族函数（全局 / 数据派生 / 项目级）；`mkdir` 副作用（`28/50/67/105`）；`get_project_config_dir` 由 `cwd` 决定，与 `OPENHARNESS_CONFIG_DIR` 无关
- 要能回答：为什么"读一次配置"会在我当前目录里创建 `.openharness/`？

### 2. `config/__init__.py`（34 行，扫过）
- [ ] 读完
- 重点：对外暴露的公共符号（`Settings`、`ProviderProfile`、`load_settings`、`save_settings`、四个路径 getter）
- 要能回答：外部模块应该从哪导入配置 API？

### 3. `config/schema.py`（119 行，扫过）
- [ ] 读完
- 重点：这是**通道适配器兼容层**（`_CompatModel` + `extra="allow"`），与主配置系统独立演进
- 要能回答：为什么这里有第二套 `Config` 模型？它跟 `Settings` 是什么关系？

### 4. `settings.py:42-150`（子模型）
- [ ] 读完
- 重点：`PathRuleConfig`、`PermissionSettings`、`MemorySettings`、`Sandbox*Settings`、`WebSettings`、`ProviderProfile`、`ResolvedAuth`
- 要能回答：`ProviderProfile` 的 `last_model` 与 `default_model` 有什么区别？`credential_slot` 是干什么的？

### 5. `settings.py:196-290`（内置 profile 目录）
- [ ] 读完
- 重点：`default_provider_profiles()` 的 11 个内置 profile；`auth_source` 与 `base_url` 的默认组合
- 要能回答：哪些 profile 没有 `base_url`，回落到什么端点？（s02 会回来改这里）

### 6. `settings.py:566-620`（`Settings` 本体）
- [ ] 读完
- 重点：33 个顶层字段与默认值，对照笔记第 1 节的字段表逐行核对
- 要能回答：哪几个字段是嵌套模型而非标量？默认姿态里哪三条最影响安全？

### 7. `settings.py:629-700`（profile 投影，最绕的一段）
- [ ] 读完
- 重点：`resolve_profile()`、`materialize_active_profile()`、`sync_active_profile_from_flat_fields()`、`resolve_auth()`
- 要能回答：一个"扁平字段"什么时候会写回 profile？`permission.mode.value` 出现在哪一行（步骤 7 的崩溃点）？

### 8. `settings.py:928-1040`（环境变量覆盖层）
- [ ] 读完
- 重点：`_apply_env_overrides()`；"专有变量无条件覆盖 / 原生变量有条件覆盖"的分支；`_parse_bool_env()` 的取值集合
- 要能回答：为什么设了 `ANTHROPIC_MODEL` 却不生效？（找出决定胜负的那一行）

### 9. `settings.py:1047-1104`（读与写）
- [ ] 读完
- 重点：`load_settings()` 的合并顺序；`save_settings()` 的 profile 同步 + 文件锁 + 原子写
- 要能回答：为什么写配置需要文件锁？崩溃发生在锁之前还是之后？

### 10. `cli.py:1975-2035`（CLI 读写入口）
- [ ] 读完
- 重点：`_config_resolve_target()` 的点号键解析；`_config_coerce_value()` 支持的 5 种类型；`config show` / `set` 的实现
- 要能回答：枚举字段为什么不在支持列表里，会导致什么后果？

### 11. `commands/registry.py:266-327`、`1135-1152`（脱敏与 TUI 入口）
- [ ] 读完
- 重点：`_SECRET_KEY_PARTS` 的子串匹配（裸词 `token` / `credential`）；`_settings_json_for_display()`；`_coerce_setting_value()` 与 `/config` 处理器
- 要能回答：两套写入口的实现差在哪？分叉点在哪个字段类型上？

## 三、必答三问（读完合上源码回答）

1. 一个配置值从磁盘到运行时，经过哪几个函数、在哪一步被 profile 覆盖？
2. 优先级是在**哪一行**决定谁赢的？给出代码位置，而不是描述。
3. `config set` 写一个值时，哪些类型安全、哪些会崩？要能指到具体行。

## 四、附加来源：`tests/test_config/`

测试是上游对"什么行为算契约"的书面声明，比注释权威。重点看：环境变量优先级的断言、坏配置的容错断言、profile 切换的断言。

- [ ] 读完 `tests/test_config/`

## 五、读完的自检

- [ ] 能画出 `settings.json → load_settings() → _apply_env_overrides() → 运行时` 的链路，并标出每层谁覆盖谁
- [ ] 能说出"三个配置视图"（文件原文 / 解析后 / 运行时）各自怎么拿
- [ ] 能指出两处上游缺陷的代码位置：脱敏误伤、枚举字段崩溃

## 六、我的发现（边读边写）

| 段号 | 符号 / 行号 | 一句话（我能复述的原理） | 意外之处 |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |
| 9 | | | |
| 10 | | | |
| 11 | | | |
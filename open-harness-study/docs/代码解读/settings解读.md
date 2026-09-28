# Settings.py 代码解读

- 源文件：`D:\code\OhAgent\.refs\OpenHarness\src\openharness\config\settings.py`
- 上游快照：v0.1.9 / `9b2efd7`
- 文件规模：1104 行

一句话总结：**它是 OpenHarness 的配置中心**——用 Pydantic 定义所有可配置项，负责把「默认值 → `settings.json` → 环境变量 → CLI 参数」合并成一个 `Settings` 对象，并处理 provider profile、旧版扁平字段兼容、API Key / 订阅鉴权解析，以及配置的原子读写。

---

## 一、整体数据流

```text
Settings() 内置默认值
   ↓
读取 ~/.openharness/settings.json（存在则 model_validate）
   ↓
OPENHARNESS_PROFILE 指定 active_profile
   ↓
若文件缺少 profiles/active_profile → 把旧版扁平字段迁移成 profile
   ↓
materialize_active_profile()：profile → 顶层扁平字段
   ↓
_apply_env_overrides()：环境变量覆盖
   ↓
CLI 层再调用 merge_cli_overrides()
   ↓
resolve_auth() / resolve_api_key() 解析凭据
```

文件开头 1–8 行的 docstring 声明优先级为：

1. CLI 参数
2. 环境变量
3. 配置文件
4. 默认值

注意：这是**整个程序**的优先级。`load_settings()` 本身只做「文件 → 环境变量」；CLI 覆盖发生在后面的 `merge_cli_overrides()`。

---

## 二、子配置模型（43–122 行）

| 类 | 作用 |
|---|---|
| `PathRuleConfig` | 单条 glob 路径权限规则，`pattern` + `allow` |
| `PermissionSettings` | 权限模式、允许/拒绝工具、路径规则、拒绝命令 |
| `MemorySettings` | 记忆文件数量/大小、上下文窗口、自动压缩、自动提取、session memory、auto dream |
| `SandboxNetworkSettings` | 沙箱网络白名单/黑名单域名 |
| `SandboxFilesystemSettings` | 沙箱读写路径规则，默认允许写 `.` |
| `DockerSandboxSettings` | Docker 镜像、CPU/内存限制、额外挂载和环境变量 |
| `SandboxSettings` | 沙箱总开关、后端、不可用是否失败、网络/文件系统/Docker 子配置 |
| `WebSettings` | 出站 web 工具的代理、DNS 解析模式、合成 DNS CIDR |

这些模型大量使用 `Field(default_factory=...)`，避免可变默认值共享。

---

## 三、Provider 与鉴权体系（124–507 行）

### 1. `ProviderProfile`（124–146 行）

描述一个「命名 provider 工作流」：

```python
label                          # 展示名
provider                       # anthropic / openai / openai_codex / ...
api_format                     # anthropic / openai / copilot
auth_source                    # anthropic_api_key / codex_subscription / ...
default_model                  # 默认模型
base_url                       # 自定义端点
last_model                     # 用户上次选择的模型
credential_slot                # 自定义 profile 独立凭据槽
allowed_models                 # 可选模型列表
context_window_tokens          # 上下文窗口配置
auto_compact_threshold_tokens  # 自动压缩阈值
```

`resolved_model` 属性（139–146 行）：实际模型 = `last_model` 优先，否则 `default_model`，再交给 `resolve_model_setting()` 转成运行时具体模型 ID。

### 2. `ResolvedAuth`（149–157 行）

冻结 dataclass，表示解析完成的鉴权结果：

```python
provider, auth_kind, value, source, state
```

`auth_kind` 可能是 `api_key`、`oauth`、`oauth_device`；`source` 用于说明凭据来源。

### 3. Claude 模型别名（160–193 行）

- `CLAUDE_MODEL_ALIAS_OPTIONS`：UI 可选别名，如 `default`、`best`、`sonnet`、`opus`、`haiku`、`sonnet[1m]`、`opusplan`。
- `_CLAUDE_ALIAS_TARGETS`：别名到具体模型 ID 的映射，如 `sonnet → claude-sonnet-4-6`。
- `normalize_anthropic_model_name()`：去掉 `anthropic/` 前缀，把 Claude 版本号中的 `.` 换成 `-`。

### 4. `resolve_model_setting()`（315–353 行）

把用户可见的模型设置解析成运行时模型 ID：

- 空值或 `default` → 用传入的 `default_model`；没有则 Claude 返回 sonnet，其它返回 `gpt-5.4`
- Claude 系：`best` → opus；`opusplan` 在 plan 模式 → opus，否则 sonnet；其它别名查表
- OpenAI 系：`best` → `gpt-5.4`

### 5. 内置 provider 目录（196–287 行）

`default_provider_profiles()` 每次返回一份新的内置配置，包含：

`claude-api`、`claude-subscription`、`openai-compatible`、`codex`、`copilot`、`moonshot`、`gemini`、`minimax`、`nvidia`、`qwen`、`modelscope`

每个都定义了 provider、API 格式、鉴权来源、默认模型和必要的 base_url。

配套的 `builtin_provider_profile_names()` 返回名称集合；`display_label_for_profile()` 对内建 profile 强制使用当前内置 label，避免旧配置文件里的过时文案残留。

### 6. 鉴权辅助函数（356–443 行）

| 函数 | 作用 |
|---|---|
| `auth_source_provider_name()` | 鉴权来源 → 存储/运行时 provider 名 |
| `auth_source_uses_api_key()` | 是否以 `_api_key` 结尾 |
| `auth_source_env_var_candidates()` | 返回该来源应探测的环境变量，`OPENHARNESS_*` 优先，原生变量其次 |
| `resolve_auth_env_value()` | 按顺序找到第一个非空环境变量 |
| `credential_storage_provider_name()` | 决定凭据存储命名空间；自定义 profile 可用 `profile:<credential_slot>` 隔离 |
| `default_auth_source_for_provider()` | provider/API 格式 → 默认鉴权来源 |

### 7. 旧版扁平字段 → profile 迁移（446–506 行）

- `_slugify_profile_name()`：把非法字符转成 `-`，生成合法 profile 名。
- `_infer_profile_name_from_flat_settings()`：根据 `provider`、`api_format`、`base_url` 推断 profile 名。
- `_profile_from_flat_settings()`：能匹配内置 profile 就复制并写入 `last_model`；否则构造一个 `Imported <provider>` 自定义 profile。

这是为了兼容旧版 `settings.json` 没有 `profiles` 字段的情况。

---

## 四、专用配置（509–563 行）

- `ImageGenerationConfig`：图片生成 provider、模型、API Key、base_url、Codex 专用配置；`from_env()` 读取 `OPENHARNESS_IMAGE_GENERATION_*`；`is_configured` 判断是否可用。
- `VisionModelConfig`：多模态 fallback 模型；`from_env()` 读取 `OPENHARNESS_VISION_*`；需要 model 和 api_key 同时存在。

---

## 五、`Settings` 主体（566–612 行）

字段按用途分组：

| 分组 | 字段 |
|---|---|
| API | `api_key`、`model`、`max_tokens`、`base_url`、`timeout`、上下文窗口、压缩阈值、`api_format`、`provider`、`active_profile`、`profiles`、`max_turns` |
| 行为 | `system_prompt`、`permission`、`hooks`、`memory`、`sandbox`、`web`、插件/技能开关、`mcp_servers` |
| UI | `theme`、`output_style`、`vim_mode`、`voice_mode`、`fast_mode`、`effort`、`passes`、`verbose` |
| 多模态 | `vision`、`image_generation` |

这里同时存在**两套表示**：

- 扁平字段：`provider`、`api_format`、`base_url`、`model`、`context_window_tokens` 等（旧接口）
- profile 字段：`active_profile` + `profiles`（新接口）

两者通过下面两个方法互相投影。

---

## 六、关键方法逐个解释

### 1. `merged_profiles()`（614–627 行）

以内置 provider 目录为底，叠加用户保存的 `self.profiles`。

- 用户 profile 是深拷贝，防止意外修改原对象。
- 如果用户 profile 没有 `base_url`，而内置 profile 有，则用内置值补齐。
- 返回「内置 + 用户覆盖」的完整字典。

### 2. `resolve_profile()`（629–637 行）

决定当前生效的 profile：

1. 显式传入的 `name`
2. `self.active_profile`
3. 环境变量 `OPENHARNESS_PROFILE`
4. 默认 `claude-api`

如果 profile 名不在合并后的目录里，就调用 `_profile_from_flat_settings()` 构造一个 fallback profile，并把它加入返回结果。返回 `(profile_name, profile深拷贝)`。

### 3. `materialize_active_profile()`（639–659 行）

把当前 profile **反向展开**到旧版扁平字段：

- `provider`、`api_format`、`base_url`
- `context_window_tokens`、`auto_compact_threshold_tokens`
- `model`：把 profile 的模型设置通过 `resolve_model_setting()` 转成具体运行时模型 ID，并传入当前权限模式（因此 `opusplan` 能按 plan/default 正确解析）

它是「profile 层 → 扁平字段层」的桥。

### 4. `sync_active_profile_from_flat_fields()`（661–722 行）

反向操作：把旧版扁平字段**折回**当前 profile。

- 如果是从 `OPENHARNESS_PROFILE` 指定 profile，则直接采用目标 profile 的 provider/api_format/base_url，避免把旧 profile 的扁平字段带过去。
- 否则比较扁平字段和 profile 字段是否一致，决定采用哪一边。
- 处理 model：扁平 model 与 profile 解析结果不同时，以扁平 model 为准；否则保留 `profile.last_model`。
- 若 auth_source 为空或等于旧默认值，则根据新的 provider/api_format 重新推断。
- 最后把更新后的 profile 写回 `profiles[profile_name]`。

它是保存配置和 CLI override 时的兼容层。

### 5. `resolve_api_key()`（724–757 行）

用于「普通 API Key」调用路径：

1. `openai_codex` → 转交给 `resolve_auth()`
2. `anthropic_claude` → 直接抛错（订阅模式必须走 `resolve_auth()`）
3. `copilot` → 返回占位符 `"copilot-managed"`
4. `self.api_key` 优先
5. 再查 profile 对应的环境变量
6. 都没有 → `ValueError`

注意这里的顺序是 **扁平 api_key > 环境变量**。

### 6. `resolve_auth()`（759–866 行）

统一的运行时鉴权解析器，比 `resolve_api_key()` 更完整：

1. **订阅桥接**：`codex_subscription` / `claude_subscription`
   - `claude_subscription` 且存在 `ANTHROPIC_AUTH_TOKEN` → 直接作为 OAuth token
   - Claude 订阅只允许官方/直连端点，第三方 base_url 会抛错
   - 否则加载外部绑定，找不到就提示先运行 `oh auth codex-login` 或 `oh auth claude-login`
   - Claude 订阅会按需刷新 token
2. **Copilot OAuth** → 返回 `copilot-managed`
3. **profile 专属凭据槽**：先查 keyring，再查文件，命名空间为 `profile:<credential_slot>`
4. **环境变量**：按 `OPENHARNESS_* → 原生变量` 顺序查找
5. **扁平 `self.api_key`**：仅在 profile 没有 `credential_slot` 时使用
6. **存储凭据**：keyring / 文件
7. 都没有 → 抛错

注意与 `resolve_api_key()` 不同：`resolve_auth()` 是 **环境变量 > 扁平 api_key**，因为设计上认为环境变量代表当前 provider 的正确凭据，而配置里的扁平字段可能残留其它 provider 的旧 key。

### 7. `merge_cli_overrides()`（868–925 行）

把 CLI 覆盖合并成新的 `Settings`：

- 忽略 `None`，只覆盖非空值
- `permission_mode` 映射到嵌套的 `permission.mode`
- `model` 会剥离 ANSI 转义序列，防止终端格式污染 API 请求
- `effort` 为 `max` 时归一化为 `xhigh`
- 涉及 profile 的覆盖（model/base_url/api_format/provider/api_key/active_profile/profiles/context_window/compact_threshold）时，调用 `sync_active_profile_from_flat_fields().materialize_active_profile()` 保持两层一致
- 如果同时切换 `active_profile` 又覆盖了 model 等字段：先切到目标 profile 并 materialize，确保不继承旧 profile 的扁平字段，再应用剩余覆盖并重新同步

---

## 七、环境变量覆盖（928–1039 行）

`_apply_env_overrides()` 的核心规则：

- `OPENHARNESS_*` 变量代表用户显式意图，**总是覆盖**
- provider 原生变量（如 `ANTHROPIC_MODEL`、`ANTHROPIC_BASE_URL`、`OPENAI_BASE_URL`）只在 profile 没有显式配置对应字段时生效
- `ANTHROPIC_BASE_URL` 优先于 `OPENAI_BASE_URL`

覆盖项包括：

| 环境变量 | 作用 |
|---|---|
| `OPENHARNESS_MODEL` / `ANTHROPIC_MODEL` | 模型 |
| `OPENHARNESS_BASE_URL` / `ANTHROPIC_BASE_URL` / `OPENAI_BASE_URL` | 端点 |
| `OPENHARNESS_MAX_TOKENS` / `OPENHARNESS_TIMEOUT` / `OPENHARNESS_MAX_TURNS` | 请求与轮次限制 |
| `OPENHARNESS_CONTEXT_WINDOW_TOKENS` / `OPENHARNESS_AUTO_COMPACT_THRESHOLD_TOKENS` | 上下文与压缩 |
| `OPENHARNESS_PROVIDER` / `OPENHARNESS_API_FORMAT` | provider 与 API 格式 |
| `OPENHARNESS_<PROVIDER>_API_KEY` / 原生 key | API Key |
| `OPENHARNESS_SANDBOX_*` | 沙箱开关、后端、Docker 镜像等 |
| `OPENHARNESS_WEB_*` | web 代理、解析模式、合成 DNS |

`_parse_bool_env()`（1042–1044 行）：`1/true/yes/on` 视为真。

---

## 八、加载与保存（1047–1104 行）

### `load_settings(config_path=None)`（1047–1083 行）

1. 未指定路径时，调用 `get_config_file_path()`，默认是 `~/.openharness/settings.json`，可用 `OPENHARNESS_CONFIG_DIR` 改目录
2. 文件存在：读取 JSON → `Settings.model_validate()`
3. `OPENHARNESS_PROFILE` 覆盖 `active_profile`
4. 若 JSON 里没有 `profiles` 或 `active_profile`，执行旧版扁平字段迁移
5. `materialize_active_profile()` → `_apply_env_overrides()`
6. 文件不存在：用默认 `Settings()`，同样处理 env profile 和环境变量

### `save_settings(settings, config_path=None)`（1086–1104 行）

1. 未指定路径时用默认路径
2. `sync_active_profile_from_flat_fields()` 把扁平字段折回 profile
3. `materialize_active_profile()` 再展开回扁平字段，确保两层一致
4. 使用 `<settings.json>.lock` 加独占文件锁
5. `atomic_write_text()` 原子写入格式化 JSON

这样可以在并发保存时避免读到半截文件。

---

## 九、值得注意的设计点与坑

1. **双层配置是最大特点**
   旧代码读扁平字段，新代码读 profile。任何直接修改 `Settings(provider=..., model=...)` 的调用，都应该先 `sync_active_profile_from_flat_fields()` 再 `materialize_active_profile()`，否则两层可能不一致。

2. **`resolve_api_key()` 与 `resolve_auth()` 的优先级不同**
   `resolve_api_key()`：扁平 `api_key` > 环境变量；
   `resolve_auth()`：环境变量 > 扁平 `api_key`。
   订阅、credential_slot、keyring 等只有 `resolve_auth()` 支持。

3. **`ProviderProfile.resolved_model` 不传 permission_mode**
   属性内部调用 `resolve_model_setting(..., permission_mode=None)`，所以 `opusplan` 在属性路径下会解析为 sonnet；只有 `materialize_active_profile()` 传入真实权限模式。直接使用该属性时要留意这一点。

4. **环境变量覆盖只改扁平字段**
   `_apply_env_overrides()` 把 env 值写入扁平字段，不会立刻改 `profiles[active].last_model`。下一次 `save_settings()` 才会折回 profile。

5. **`credential_slot` 的隔离机制**
   自定义 OpenAI 兼容 profile 可以设置 `credential_slot`，凭据存到 `profile:<slot>`；内建 API-key profile 仍使用 provider 级命名空间。这允许同一 provider 配置多把 key。

6. **小段冗余/未覆盖路径**
   - `resolve_auth()` 第 812 行的 `storage_provider` 赋值会在第 830 行被覆盖，实际未生效。
   - `resolve_model_setting()` 第 350 行的 `{"default", "best"}` 中，`default` 分支在前面的早返回中已处理，属于不可达。
   - `bedrock_api_key` / `vertex_api_key` 没有加入 `auth_source_env_var_candidates()`，这两个来源无法通过该函数走环境变量解析。

7. **⚠️ 需要特别留意的安全点：环境变量中的 API Key 可能被写回配置文件**
   `_apply_env_overrides()` 在 991–994 行把解析到的 API Key 写入扁平字段 `settings.api_key`；`save_settings()` 在 1098–1104 行直接把整个模型序列化落盘。
   也就是说，如果通过 `OPENHARNESS_ANTHROPIC_API_KEY` 等环境变量提供密钥，之后又触发了任意 `save_settings()`（例如 `/theme`、`/model`、`/effort` 这类命令都是 `load_settings()` → 修改 → `save_settings()`），该密钥就会被写入 `settings.json`。
   我用临时目录实际验证过该行为：设置 `OPENHARNESS_ANTHROPIC_API_KEY` 后执行 `load_settings()` + `save_settings()`，生成的 JSON 中包含该 key。这与仓库「密钥只通过环境变量注入、禁止提交」的安全约定存在冲突，适合单独记入研习笔记；上游快照是只读的，这里只做记录。

---

## 十、建议的阅读顺序

如果第一次读这个文件，按下面的顺序最省力：

1. 43–122 行：子配置模型
2. 124–196 行：`ProviderProfile`、`ResolvedAuth`、模型别名
3. 196–287 行：内置 provider 目录
4. 356–506 行：鉴权映射与旧版迁移
5. 566–722 行：`Settings` 主体与 profile 双向投影
6. 724–866 行：`resolve_api_key()` / `resolve_auth()`
7. 868–925 行：CLI 覆盖
8. 928–1104 行：环境变量、加载、保存

这样先建立「配置有哪些 + profile 是什么」，再看「怎么解析」和「怎么持久化」，最后回头看顶部的优先级声明，就能把整个文件串起来。
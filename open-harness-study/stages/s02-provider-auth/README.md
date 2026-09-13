# s02 · Provider 与认证体系

> 前置：s01 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 说清 OpenHarness 的 **workflow + profile** 双层模型，以及它相对"协议名"的优势。
2. 能独立接入任意企业模型网关（Anthropic 兼容 / OpenAI 兼容 / 订阅制），并完成凭据隔离。
3. 理解流式、重试、用量统计与错误分类在客户端层如何实现。

## 二、知识点清单

- **协议客户端**（`api/`）
  - `client.py`：Anthropic 客户端，含重试逻辑
  - `openai_client.py`：OpenAI 兼容层（覆盖 DashScope / OpenRouter / DeepSeek / SiliconFlow / Gemini / Groq / Ollama / NVIDIA NIM 等）
  - `codex_client.py`：Codex 订阅（走 chatgpt.com Codex Responses），支持独立的 reasoning effort 参数
  - `copilot_client.py` + `copilot_auth.py`：GitHub Copilot OAuth device flow
  - `provider.py`：provider/auth 能力判定；`errors.py`：错误类型；`usage.py`：用量模型
- **workflow 与 preset**（`api/registry.py`）：`Anthropic-Compatible API`、`Claude Subscription`、`OpenAI-Compatible API`、`Codex Subscription`、`GitHub Copilot`
- **认证体系**（`auth/`）
  - `manager.py` 统一管理器；`storage.py` 凭据存储；`flows.py` 各认证流程
  - `external.py`：复用外部 CLI 订阅凭据（`~/.claude/.credentials.json`、`~/.codex/auth.json`）
  - **credential slot**：profile 级独立凭据，避免 Kimi/GLM 等共用一把全局 key
- **CLI 面**：`oh setup`（workflow → 认证 → preset → 模型 → 激活）、`oh auth login/status/logout/switch/*-login`、`oh provider list/use/add/edit/remove`、`/provider`、`/model`
- **企业接入要点**：自定义 `base_url`、`allowed_models` 白名单、`context_window_tokens` 与压缩阈值绑定、私有网关的 TLS/代理

## 三、关键接口 / 符号

- `ProviderProfile`（`provider` / `api_format` / `auth_source` / `default_model` / `base_url` / `credential_slot` / `allowed_models`）
- `providers: dict[str, ProviderProfile]`、`active_profile`、`api_format ∈ {anthropic, openai, copilot}`
- `oh provider add <name> --label --provider --api-format --auth-source --model --base-url --credential-slot --api-key`

## 四、动手实验

```powershell
uv run oh setup
uv run oh provider list
uv run oh auth status

# 接入一个自定义企业网关 profile
uv run oh provider add corp-gw `
  --label "Corp Gateway" --provider anthropic --api-format anthropic `
  --auth-source anthropic_api_key --model claude-sonnet-4-6 `
  --base-url https://llm.corp.example/anthropic --credential-slot corp-gw

uv run oh --dry-run -p "ping" --output-format json   # 用 dry-run 验证凭据就绪
uv run oh -p "用一句话说明你是谁"
```

任务：
1. 绘制"一次模型调用"的时序图：CLI → settings → profile → auth → client → 流式返回。
2. 对比 Anthropic 兼容与 OpenAI 兼容两条链路在客户端代码上的差异（消息格式、流式事件、工具调用）。
3. 设计一份**多环境 profile 命名规范**（dev/staging/prod × 供应商）。

## 五、验收标准

- 能新增一个自定义 profile 并跑通真实调用（含自定义 `base_url`）。
- 能解释 `credential_slot` 缺失时凭据会怎么取，以及由此带来的风险。
- 能说出 5 个内置 workflow 各自适用的企业场景。
- 出现 401/429/超时时，能定位到是哪一层（认证 / 限流 / 客户端重试）并给出处置。

## 六、进阶思考

- 企业统一网关常要求"按业务线配额 + 全量审计"。OpenHarness 的客户端层缺少什么？你会加在哪？
- `api_format` 与 `provider` 为什么要分开？给出一个"同协议不同厂商"的真实例子。
- 若上游新增一家供应商，最小改动是"加 preset"还是"加 client"？依据是什么（读 `api/registry.py`）？
# s16 · Channels 与 ohmo Gateway

> 前置：s15 ｜ 建议投入：10–12h ｜ 难度：★★★★

## 一、学习目标

1. 掌握"通道 → MessageBus → QueryEngine"的解耦架构。
2. 能跑通 `ohmo`，理解工作区（人格/用户画像/引导仪式）与记忆隔离。
3. 能接入至少一个 IM 通道，并理解多通道路由与鉴权要点。

## 二、知识点清单

- **通道架构**
  - `channels/adapter.py:ChannelBridge`：把 MessageBus 与 `QueryEngine` 连起来
  - `channels/bus/queue.py`：异步消息队列；`channels/bus/events.py`：事件类型
  - `channels/impl/base.py`：通道基类（收发、媒体、错误）
  - `channels/impl/manager.py`：通道管理器
- **已实现的通道**（`channels/impl/`）：`telegram` `slack` `discord` `feishu`（lark-oapi 长连接，最大实现，约 54 KB） `dingtalk` `qq` `email`（IMAP + SMTP） `matrix` `whatsapp`（Node bridge） `mochat`（Socket.IO）
- **安全关注**：`tests/test_channels/test_feishu_security.py`、`test_telegram_security.py`——通道鉴权、签名校验、防重放
- **ohmo 个人智能体**（`ohmo/`）
  - 工作区 `~/.ohmo/`：`soul.md`（人格）`identity.md`（自我认知）`user.md`（用户画像）`BOOTSTRAP.md`（首次引导）`memory/`（个人记忆，**与项目记忆隔离**）`gateway.json`
  - `ohmo/cli.py`：`init` / `config` / 运行；`ohmo/workspace.py`、`prompts.py`、`memory.py`、`session_storage.py`、`group_registry.py`
  - `ohmo/gateway/`：`service.py`（服务）`router.py`（路由）`bridge.py` `notify.py` `config.py` `models.py` `provider_commands.py` `group_tool.py`
- **命令**：`ohmo init` / `config` / `ohmo` / `ohmo gateway run|status|restart`；`ohmo config` 复用与 `oh setup` 一致的 workflow 语言，并引导配置 Telegram/Slack/Discord/Feishu
- **企业议题**：多租户、会话归属、群聊 vs 私聊、消息去重、长消息分段、富媒体、速率限制

## 三、关键接口 / 符号

- `ChannelBridge`、`MessageBus`（queue/events）、`ChannelManager`
- `ohmo gateway run|status|restart`、`~/.ohmo/gateway.json`

## 四、动手实验

```powershell
uv run ohmo init
uv run ohmo config
uv run ohmo gateway run
# 另开终端
uv run ohmo gateway status
```

任务：
1. 配置 Telegram（或飞书）通道，实现"发消息 → Agent 执行 → 回复"闭环。
2. 触发一次工具调用，观察中间进度/工具提示是否回传（参考 CHANGELOG 中 Telegram 回复回归问题的修复点）。
3. 读 `ChannelBridge`，画出"消息进来 → 队列 → 引擎 → 输出 → 发回"的路径。
4. 实现一个最小自定义通道（如内部 webhook），接入 MessageBus。
5. 设计多租户路由：如何把"不同群/不同企业"映射到不同配置与会话。

## 五、验收标准

- 能解释为什么需要 MessageBus 而不是让通道直接调用引擎。
- 至少一个 IM 通道端到端可用，且长任务有中间反馈。
- 能说清 ohmo 的个人记忆为什么必须与项目记忆隔离。
- 能列出通道层的安全清单（签名校验、鉴权、重放、注入、限流）。

## 六、进阶思考

- 通道是"长连接 + 有状态"的；网关重启后如何保证消息不丢、不重复？
- 群聊场景下多个用户同时对话，会话与会话锁如何设计？
- 通道输出受平台限制（长度、格式、速率）。渲染层需要怎样的适配抽象？
- 若企业已有统一消息中台，是"写一个新 channel"还是"接中台 webhook"？
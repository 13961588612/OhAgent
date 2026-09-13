# s17 · TUI 与前端协议

> 前置：s16 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 理解两套 UI 并存的设计：Textual TUI 与 React/Ink 终端前端。
2. 掌握前端与后端之间的 **JSON-lines 协议**，能扩展协议字段。
3. 掌握主题、键位、Vim、语音、输出风格等体验层的定制方式。

## 二、知识点清单

- **UI 装配**：`ui/runtime.py:build_runtime`（约 32 KB）——headless 与 TUI 共用同一运行时
- **Textual UI**：`ui/textual_app.py`（约 18 KB）——默认终端界面
- **React 终端前端**：`ui/react_launcher.py` 启动；后端为 `ui/backend_host.py`
- **协议**：`ui/protocol.py`（结构化消息模型）、`ui/backend_host.py`（JSON-lines 后端宿主）
- **渲染**：`ui/output.py`（rich markdown、语法高亮、spinner）、`ui/permission_dialog.py`（权限弹窗，含并发串行化锁）、`ui/coordinator_drain.py`（协调者模式在回合间排空后台任务）
- **前端代码**（`frontend/terminal/src/`）
  - `App.tsx`、`index.tsx`、`types.ts`、`clipboardImage.ts`
  - 组件：`Composer` `ConversationView` `TranscriptPane` `ToolCallDisplay` `StatusBar` `SidePanel` `SwarmPanel` `TodoPanel` `CommandPicker` `SelectModal` `ModalHost` `MarkdownText` `PromptInput` `Spinner` `WelcomeBanner` `Footer`
  - `hooks/useBackendSession.ts`、`theme/`（`builtinThemes.ts`、`ThemeContext.tsx`）
- **体验配置**：`themes/`（内置主题 + loader + schema）、`keybindings/`（default_bindings/loader/parser/resolver）、`vim/`、`voice/`（`voice_mode.py`、`keyterms.py`、`stream_stt.py`）、`output_styles/`
- **已知体验细节**（来自 CHANGELOG，读它可理解真实工程问题）：DEL 字节退格、退出后 shell 提示符粘连、Markdown 表格对齐、双击 Enter 重复提交、spinner 生命周期
- **测试**：`tests/test_ui/*`（含 `test_react_backend.py`、`test_react_launcher.py`、`test_project_plugin_security.py`）、`scripts/react_tui_e2e.py`、`scripts/test_tui_interactions.py`

## 三、关键接口 / 符号

- `build_runtime(...)`、`BackendHostConfig`、`ui/protocol.py` 中的消息类型
- `/theme` `/keybindings` `/vim` `/voice` `/output-style`

## 四、动手实验

```powershell
uv run oh
uv run pytest tests/test_ui -q
uv run python scripts/test_headless_rendering.py
```

任务：
1. 用 `/theme`、`/keybindings` 定制一套团队默认外观与键位，并说明如何随企业配置下发。
2. 读 `ui/protocol.py`，为"工具调用的实时耗时"新增一个协议字段，并让前端展示。
3. 复现一个真实体验问题（如长 Markdown 表格错位），定位到渲染层并修复。
4. 评估：headless `-p` 与 TUI 在事件消费上的差异，为什么可以共用同一运行时？

## 五、验收标准

- 能画出 "前端 → backend_host → runtime → engine" 的交互与协议格式。
- 能说出 `backend_host` 为什么用 JSON-lines（相对于 WebSocket/gRPC 的取舍）。
- 至少完成一次前端或协议的定制改动，并有运行截图/输出证据。
- 能解释权限弹窗为何需要串行化锁。

## 六、进阶思考

- 两套 UI 并存带来维护成本。企业只保留一套时该选哪个？判据是什么？
- 前端如何展示"多 Agent 协作"（`SwarmPanel`）而不至于信息过载？
- 如果要替换为企业自研 Web UI，最小改动面在哪（协议层是否足够稳定）？
- 终端 UI 的可访问性（无障碍、配色对比）企业合规上是否需要考虑？
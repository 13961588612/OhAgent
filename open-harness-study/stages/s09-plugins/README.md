# s09 · Plugins 插件体系

> 前置：s08 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握 `plugin.json` 清单与插件可贡献的六类产物。
2. 理解插件的发现路径与**信任边界**（为什么项目级插件默认禁用）。
3. 能把"工具 + 技能 + 命令 + 子 Agent + 钩子 + MCP"打包成一个企业插件并分发。

## 二、知识点清单

- **清单**（`plugins/schemas.py:PluginManifest`）
  - `name` `version` `description` `enabled_by_default`
  - `skills_dir`（默认 `skills`）、`tools_dir`（默认 `tools`）、`hooks_file`（默认 `hooks.json`）、`mcp_file`（默认 `mcp.json`）
  - 扩展字段：`author` `commands` `agents` `skills` `hooks`
  - 兼容布局：`plugin.json` 或 `.claude-plugin/plugin.json`
- **发现路径**（`plugins/loader.py`）
  - 用户级：`~/.openharness/plugins`
  - 项目级：`<cwd>/.openharness/plugins`（需 `allow_project_plugins=true`，否则仅告警并在 UI 测试中受安全校验）
- **可贡献产物**
  1. Skills（`<plugin>/skills/*/SKILL.md`，或插件根 `SKILL.md`）
  2. Tools（`<plugin>/tools/` 下的 `BaseTool` 子类，运行时自动发现、实例化、注册）
  3. Commands（`PluginCommandDefinition`：名称、描述、内容、参数提示、模型、effort）
  4. Agents（AgentDefinition，见 s13）
  5. Hooks（`hooks.json`）
  6. MCP（`mcp.json`）
- **启用控制**：`settings.enabled_plugins: dict[str, bool]`；`installer.py` 负责安装
- **CLI/斜杠**：`oh plugin list|install|uninstall`、`/plugin`、`/reload-plugins`
- **安全**：项目级插件的默认禁用是"防投毒"设计；插件工具与内置工具同权限通道

## 三、关键接口 / 符号

- `PluginManifest`、`LoadedPlugin`、`PluginCommandDefinition`
- `load_plugins(settings, cwd, extra_roots)`、`get_user_plugins_dir()`、`get_project_plugins_dir(cwd)`
- 开关：`allow_project_plugins`、`enabled_plugins`

## 四、动手实验

```powershell
uv run oh -p "/plugin list"
uv run oh plugin list
```

任务：
1. 手工搭一个企业插件 `corp-toolkit`，包含：1 个工具、1 个技能、1 个斜杠命令、`hooks.json`（审计）、`mcp.json`（内部系统）。
2. 安装到 `~/.openharness/plugins`，验证六类产物全部生效（用 `/plugin`、`/skills`、`/hooks`、`/mcp` 交叉验证）。
3. 把它放到项目 `.openharness/plugins`，验证 `allow_project_plugins=false` 时的行为，再打开验证。
4. 设计插件版本与升级策略（向后兼容、清单变更、禁用开关）。

## 五、验收标准

- 能说出插件可贡献的六类产物及其对应目录/文件。
- 自研插件能在真实会话中同时提供工具与技能，且钩子生效。
- 能解释 `enabled_by_default` 与 `enabled_plugins` 的优先级。
- 能说清"项目级插件默认禁用"的安全动机，并给出企业分级信任方案。

## 六、进阶思考

- 插件工具直接跑在宿主进程里，等于**任意代码**。企业要分发第三方插件需要什么（签名、沙箱、审计、白名单）？
- 插件与容器化/包管理（pip、npm）如何协同？如何保证可复现部署？
- 若同一插件由多个来源提供（用户级 + 项目级 + 插件市场），冲突与隔离如何设计？
- 插件贡献的 MCP 与用户自己的 `settings.mcp_servers` 冲突时，谁优先？依据哪段源码？
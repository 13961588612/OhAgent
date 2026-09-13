# s06 · 权限与安全治理

> 前置：s05 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握三种权限模式（`default` / `plan` / `full_auto`）的语义与切换方式。
2. 理解权限判定链：工具白/黑名单 → 路径规则 → 命令黑名单 → 交互确认。
3. 能为企业设计"最小权限 + 可审计"的落地策略。

## 二、知识点清单

- **权限模式**（`permissions/modes.py`）：`default`（默认确认）、`plan`（只读规划，禁止改动）、`full_auto`（自动放行）
- **判定器**（`permissions/checker.py`）：综合 settings 与运行时上下文给出允许/拒绝/需确认
- **配置面**（`settings.py:PermissionSettings`）
  - `allowed_tools` / `denied_tools`：工具级
  - `path_rules: [PathRuleConfig(pattern, allow)]`：路径级
  - `denied_commands`：shell 命令级
- **交互确认**：`ui/permission_dialog.py`；TUI 中并发工具调用用 `asyncio.Lock` 串行化弹窗（避免互相覆盖）
- **计划模式工具**：`enter_plan_mode_tool` / `exit_plan_mode_tool`；`/plan`、`/permissions`
- **出网安全**：`utils/network_guard.py` 做目标校验（SSRF 防护）
- **路径校验**：`sandbox/path_validator.py`
- **审计线索**：谁在什么模式下执行了什么工具、结果如何——结合 s07 hooks 落地
- **企业风险点**：`bash` + `file_write` 的组合等于任意代码执行；`web_fetch` 可能被用于数据外泄

## 三、关键接口 / 符号

- `PermissionMode.{DEFAULT, PLAN, FULL_AUTO}`
- `PermissionSettings.{mode, allowed_tools, denied_tools, path_rules, denied_commands}`
- `/permissions`、`/plan`、Tab 打开模式选择器

## 四、动手实验

```powershell
uv run oh -p "/permissions"
uv run oh -p "/plan"      # 进入计划模式后尝试写文件，观察被拒
uv run oh -p "/permissions full_auto"
```

任务：
1. 设计并落地一份企业权限基线（示例：默认 `default`；`denied_commands` 覆盖 `rm -rf /`、`curl | sh`；`path_rules` 禁止写 `/etc`、`~/.ssh`）。
2. 验证每条规则都能拦住预期操作（写测试用例）。
3. 对比 `plan` 模式与"只给只读工具"的实现差异，说明各自的可靠性。
4. 给出"高危工具分级表"（只读 / 低危写 / 高危执行 / 网络出口）。

## 五、验收标准

- 能画出权限判定顺序，并指出每一步的源码位置。
- 能解释 `denied_tools` 与 `denied_commands` 的边界（前者管工具，后者管 shell 内容），以及绕过风险。
- 产出一份可执行的企业权限基线配置（含测试证据）。
- 能说明 `full_auto` 在什么场景下是**可以接受**的（如沙箱 + 无人值守 + 只读仓库）。

## 六、进阶思考

- 基于黑名单的 `denied_commands` 天然可被绕过（编码、别名、脚本）。要真正做到 enterprise-grade 需要什么（容器 + 网络策略 + 只读挂载 + 审计）？
- 权限确认是"同步阻塞 UI"的；在 IM 通道或无人值守场景如何降级（策略预设 + 事后审计）？
- 子 Agent / 队友的权限如何继承与同步（预告：s14 `swarm/permission_sync.py`）？
- 若要求"高危操作双人复核"，现有插点是否够用？不够该加在 hooks 还是 permissions？
# s14 · Swarm 协作运行时

> 前置：s13 ｜ 建议投入：12–14h ｜ 难度：★★★★★

## 一、学习目标

1. 掌握 Swarm 的四种执行后端及其适用场景。
2. 掌握队友之间的消息邮箱、权限同步与团队生命周期。
3. 能用 git worktree 做并行 Agent 的代码隔离，避免互相踩踏。

## 二、知识点清单

- **后端类型**（`swarm/types.py`）：`subprocess` / `in_process` / `tmux` / `iterm2`（后两者为"窗格后端"，可可视化）
- **后端注册**（`swarm/registry.py:BackendRegistry` / `get_backend_registry()`）
- **执行后端**：`swarm/subprocess_backend.py`、`swarm/in_process.py`
- **消息邮箱**（`swarm/mailbox.py`）：`TeammateMailbox`、`MailboxMessage`、`create_user_message`、`create_idle_notification`、`create_shutdown_request`、`get_team_dir` / `get_agent_mailbox_dir`
- **权限同步**（`swarm/permission_sync.py`，约 39 KB）：`SwarmPermissionRequest` / `SwarmPermissionResponse`，`create/send/poll/handle_permission_request`；主 Agent 为队友做权限裁决；`_READ_ONLY_TOOLS` 用**实际注册的 snake_case 工具名**做只读自动放行
- **团队生命周期**（`swarm/team_lifecycle.py`）：创建 → 运行 → 空闲通知 → 关闭请求 → 清理
- **代码隔离**（`swarm/worktree.py`）：git worktree 隔离；配合 `enter_worktree` / `exit_worktree` 工具
- **派生与加锁**：`swarm/spawn_utils.py`、`swarm/lockfile.py`
- **平台限制**：部分模块依赖 POSIX（`swarm/__init__.py` 用懒惰导入避免 Windows 直接失败）——**Windows 上要验证可用的子集**
- **任务侧联动**：`tasks/*` 把队友当后台任务管理

## 三、关键接口 / 符号

- `BackendType`、`SpawnResult`、`TeammateExecutor`、`TeammateIdentity`、`TeammateSpawnConfig`
- `TeammateMailbox`、`MailboxMessage`
- `handle_permission_request` / `poll_permission_response`
- 工具：`agent`、`send_message`、`team_create`、`team_delete`

## 四、动手实验

```powershell
uv run pytest tests/test_swarm -q     # 先看上游怎么测，再自己写
```

任务：
1. 用 `subprocess` 后端起一个 3 人小队（1 主 + 2 队友）完成"并行重构两个模块"。
2. 观察邮箱消息流转：任务分派、进度回报、空闲通知、关闭请求。
3. 让一个队友申请高危操作权限，验证主 Agent 的裁决路径。
4. 用 worktree 隔离两个队友的工作区，制造一次冲突场景，验证隔离有效。
5. 在 Windows 上验证哪些后端可用、哪些不可用，并记录替代方案。

## 五、验收标准

- 能画出"主 Agent → 队友 → 邮箱 → 权限同步 → 结果回收"的完整时序。
- 能解释四种后端的取舍（隔离性、可见性、跨平台性、调试难度）。
- 能说明 worktree 隔离解决了什么问题、又带来什么新问题（合并成本）。
- 至少一套多队友方案能稳定跑通并产出可验证结果。

## 六、进阶思考

- 多 Agent 并行的"协调成本"何时超过收益？给出你的判断阈值。
- 队友权限同步是"请求-响应"模型；如果主 Agent 忙或退出，队友会怎样？如何设计降级？
- 团队与代码仓库的关系：worktree 隔离后，谁来负责合并与冲突解决？
- 企业要"审计每个队友的每一步"，现有留痕能力缺口在哪？
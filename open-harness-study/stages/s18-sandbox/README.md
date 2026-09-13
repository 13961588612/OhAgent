# s18 · 沙箱隔离与执行安全

> 前置：s17 ｜ 建议投入：8–10h ｜ 难度：★★★★

## 一、学习目标

1. 掌握沙箱的配置模型（网络、文件系统、Docker）与可用后端。
2. 理解"路径校验 + 出网校验 + 容器隔离"三者的分工。
3. 能为企业设计"高危执行"的隔离方案（开发机 / CI / 生产）。

## 二、知识点清单

- **沙箱实现**（`sandbox/`）
  - `adapter.py`：沙箱适配（后端选择、可用性判定）
  - `docker_backend.py`：Docker 执行后端
  - `docker_image.py` + `Dockerfile`：沙箱镜像构建（`auto_build_image`）
  - `path_validator.py`：路径合法性校验
  - `session.py`：沙箱会话生命周期
- **配置**（`settings.py`）
  - `SandboxSettings`：`enabled`、`backend`（默认 `srt`）、`fail_if_unavailable`、`enabled_platforms`
  - `SandboxNetworkSettings`：`proxy`、`resolution_mode`、`synthetic_dns_cidrs`
  - `SandboxFilesystemSettings`：`allow_read` / `deny_read` / `allow_write`（默认 `["."]`）/ `deny_write`
  - `DockerSandboxSettings`：`image`（默认 `openharness-sandbox:latest`）、`auto_build_image`、`cpu_limit`、`memory_limit`、`extra_mounts`、`extra_env`
- **出网安全**：`utils/network_guard.py`（HTTP 目标校验，防 SSRF/内网探测）
- **原子写与并发**：`utils/fs.py`、`utils/file_lock.py`
- **测试**：`tests/test_sandbox/*`、`scripts/test_docker_sandbox_e2e.py`
- **与权限的关系**：权限决定"能不能做"，沙箱决定"在哪、以什么代价做"——两者互补

## 三、关键接口 / 符号

- `SandboxSettings` / `SandboxNetworkSettings` / `SandboxFilesystemSettings` / `DockerSandboxSettings`
- `fail_if_unavailable`（企业建议 `true`：宁可失败也不裸跑）
- `/doctor` 可辅助诊断环境与沙箱可用性

## 四、动手实验

```powershell
uv run oh -p "/doctor"
uv run pytest tests/test_sandbox -q
uv run python scripts/test_docker_sandbox_e2e.py    # 需要 Docker
```

任务：
1. 打开沙箱（docker 后端），让 Agent 跑一个会写文件、装依赖的任务，验证隔离生效。
2. 配置 `allow_write: ["."]` + `deny_read: ["~/.ssh", "~/.aws"]`，验证越界读写被拒。
3. 配置 `network.denied_domains`，验证 `web_fetch` 被拦。
4. 故意把 `fail_if_unavailable` 设为 `false` 并让沙箱不可用，观察"降级裸跑"的风险。
5. 设计企业三级隔离：开发机（宽松）/ CI（容器 + 只读挂载）/ 生产运维（最严 + 全审计）。

## 五、验收标准

- 能解释 `backend=srt` 与 `backend=docker` 的区别与适用平台（注意 Windows 支持矩阵）。
- 能说明三组配置（network/filesystem/docker）各拦什么、漏什么。
- 至少一套隔离配置能拦住"越界读写 + 内网访问"两类操作，并有证据。
- 能解释"权限 + 钩子 + 沙箱"三层如何叠加成纵深防御。

## 六、进阶思考

- 沙箱内执行的代码仍可能通过合法域名外泄数据；如何做 DLP/出口审计？
- 容器隔离的成本（启动延迟、镜像维护）与收益如何平衡？什么任务值得进沙箱？
- `resolution_mode` / `synthetic_dns_cidrs` 这类网络细节为何重要（DNS 重绑定攻击）？
- 若企业已有 K8s/Job 体系，是复用还是重写 `docker_backend`？
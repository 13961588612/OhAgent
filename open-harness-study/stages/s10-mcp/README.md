# s10 · MCP 集成

> 前置：s09 ｜ 建议投入：8–10h ｜ 难度：★★★

## 一、学习目标

1. 掌握 MCP 的三种传输配置（stdio / http / ws）与适用场景。
2. 理解 MCP 工具与资源如何被**动态适配**成 OpenHarness 工具。
3. 能自研一个 MCP Server，把企业内部系统安全地接入 Agent。

## 二、知识点清单

- **配置模型**（`mcp/types.py`）
  - `McpStdioServerConfig`：`command` `args` `env` `cwd`
  - `McpHttpServerConfig`：`url` `headers`
  - `McpWebSocketServerConfig`：`url` `headers`
  - 文件形态：`McpJsonConfig{mcpServers: {...}}`（插件 `mcp.json` 与项目文件共用）
- **配置来源合并**（`mcp/config.py`）：settings 的 `mcp_servers` + 插件贡献
- **连接管理**（`mcp/client.py`）：连接、列工具、列资源、读资源、错误处理
- **动态工具适配**（`tools/mcp_tool.py:McpToolAdapter`）：把远端工具映射为本地工具注册表项
- **资源工具**：`list_mcp_resources_tool`、`read_mcp_resource_tool`
- **鉴权**：`mcp_auth_tool`、`mcp/auth` 相关配置与 headers
- **CLI/斜杠**：`oh mcp list|add|remove`、`/mcp`
- **dry-run 行为**：**不会**连接 MCP server，但会校验明显错误的配置（这是排查配置的利器）
- **测试参考**：`tests/fixtures/fake_mcp_server.py`、`tests/test_mcp/{test_stdio_flow,test_http_flow,test_integration,test_client_errors}.py`

## 三、关键接口 / 符号

- `McpServerConfig`（联合类型）、`McpJsonConfig`
- `McpClientManager`（`mcp/client.py`）、`McpToolAdapter`
- `/mcp`、`oh mcp add ...`

## 四、动手实验

```powershell
uv run oh mcp list
uv run oh --dry-run -p "查询工单" --output-format json   # 看 MCP 配置是否被判定为可用
uv run pytest tests/test_mcp -q
```

任务：
1. 用 Python MCP SDK 写一个只读 MCP Server（暴露 `get_ticket(id)` 与 `list_customers()`），支持 stdio。
2. 接入 OpenHarness，验证工具在真实会话中被调用；再故意制造错误（超时/异常）观察错误传递。
3. 用 `mcp.json` 把该 Server 打包进 s09 的企业插件。
4. 为 MCP 设计企业接入规范：命名空间、只读优先、超时、凭据注入、审计字段。
5. 对比 MCP 与"直接写 Tool"的取舍（复用性、隔离性、部署复杂度、跨语言）。

## 五、验收标准

- 能写出三种传输方式的配置样例，并说明各自的网络与部署要求。
- 自研 MCP Server 能被 Agent 成功调用，且失败时错误可读、不中断会话。
- 能解释 `--dry-run` 对 MCP 的处理边界，并会用它排查配置。
- 产出企业 MCP 接入规范（含鉴权与审计）。

## 六、进阶思考

- MCP 工具是**远端不可信代码**：如何在权限层表达"这个工具属于外部系统，需要更高确认级别"？
- `args`/`env` 里可能塞密钥，配置文件的密钥管理应如何做（env 注入 vs 密钥服务）？
- 大量 MCP 工具注册后会显著增加请求体大小，schema 裁剪与按需加载该如何设计？
- MCP 资源（resources）与工具（tools）的语义差异是什么？企业知识库更适合哪种？
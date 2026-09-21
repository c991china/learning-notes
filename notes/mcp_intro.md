# MCP 入门：大模型怎么「调用工具」

MCP（Model Context Protocol）让 AI 应用以统一方式接入外部能力（搜索、数据库、文件…）。

## 三种角色
- **Host**：你用的客户端（如 WorkBuddy、Claude Desktop）
- **Client**：Host 内与某个 Server 通信的组件
- **Server**：提供具体工具/资源的一方（如 AnySearch）

## 一次调用的生命周期
1. `initialize`：握手，约定协议版本与能力
2. `tools/list`：Host 询问「你能干什么」
3. `tools/call`：Host 带着参数请求某个工具
4. Server 返回结果，Host 喂给模型

## 传输方式
- 本地：`stdio`（命令行子进程）
- 远程：`Streamable HTTP`（如 `https://api.anysearch.com/mcp`）

> 实战接入见 [@c991china/anysearch-mcp-guide](https://github.com/c991china/anysearch-mcp-guide)。

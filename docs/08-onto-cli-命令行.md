# onto-cli —— 命令行客户端模块

> 定位：二进制 `ontodb-cli`。源码规模：`main.rs` 单文件。
> 依赖：仅 clap / serde / serde_json，**不依赖任何 onto-* crate**，通过 TCP 私有协议与服务器解耦。

---

## 1. 职责与边界

onto-cli 是与 `onto-server::run_tcp_server` 配套的命令行客户端：

- **REPL 交互模式**：逐行输入查询，返回格式化结果
- **单行查询模式**：`-q "SELECT ..."` 一次执行
- **文件执行模式**：执行 `.sql` / `.ontoql` 脚本文件
- **dump / restore**：数据导出与导入

边界：**不直接访问存储引擎**，不依赖任何内部 crate；与服务器的唯一耦合是 TCP 行协议（`\n` 分隔、`\0` 结尾，见 `main.rs` 与 `server/main.rs:1115-1251`）。

---

## 2. 命令结构（`main.rs:38-108`）

`Args` / `CliCommand` 由 clap derive 定义：

| 命令/参数 | 用途 |
|-----------|------|
| 连接参数 | `--host`、`--port`（默认指向服务器 TCP 端口） |
| `-q / --query` | 单行查询 |
| `-f / --file` | 执行脚本文件 |
| `--interactive` / REPL | 交互模式 |
| `dump` / `restore` | 数据导出导入 |

---

## 3. 通信协议

CLI 与服务器走**私有 TCP 行协议**：

```
请求：查询文本 + "\n" + "\0"（消息边界）
响应：服务器按行返回结果，格式化输出
```

该协议与 HTTP / PGWire / MySQLWire 并列，是 `onto-server` 四种入口之一，其服务端实现见 `server/main.rs:1115-1251`。

---

## 4. 实现现状与边界

1. **TLS 为 TODO，AUTH 为占位符**（`onto-cli/main.rs:172-187`）；服务器 TCP 协议同样未处理 `AUTH` 命令（`server/main.rs:1160-1251`）——即 CLI 当前无真正的认证能力。
2. CLI 是纯客户端二进制，无对外库类型。
3. 与服务器通过协议字符串耦合，若服务端行协议变更需同步修改。

> 提示：需要认证/加密的生产场景应使用 HTTP（支持 API key + TLS）或 PG/MySQL wire 协议，而非 TCP CLI 协议。

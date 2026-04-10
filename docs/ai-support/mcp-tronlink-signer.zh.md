# mcp-tronlink-signer

**GitHub**: https://github.com/TronLink/mcp-tronlink-signer

将 [tronlink-signer](https://github.com/TronLink/mcp-tronlink-signer/tree/main/packages/tronlink-signer) 封装为 MCP 工具的服务器，供 Claude 及其他 AI 客户端使用。通过 TronLink 浏览器钱包对 TRON 交易进行签名，需用户在浏览器中授权确认，私钥始终留在钱包中，不会对外暴露。

## 配置

### Claude Code

```bash
claude mcp add -s user tronlink-signer -- npx mcp-tronlink-signer
```

### Claude Desktop / Cursor

```json
{
  "mcpServers": {
    "tronlink-signer": {
      "command": "npx",
      "args": ["mcp-tronlink-signer"]
    }
  }
}
```

### 从源码构建

```bash
git clone https://github.com/TronLink/mcp-tronlink-signer.git
cd mcp-tronlink-signer
pnpm install && pnpm build
```

然后指向构建产物：

```bash
claude mcp add -s user tronlink-signer -- node /path/to/packages/mcp-tronlink-signer/dist/cli.js
```

## MCP 工具

| 工具 | 说明 | 参数 |
|---|---|---|
| `connect_wallet` | 连接 TronLink 钱包 | `network?` |
| `send_trx` | 向指定地址发送 TRX | `to`、`amount`、`network?` |
| `send_trc20` | 发送 TRC20 代币 | `contractAddress`、`to`、`amount`、`decimals?`、`network?` |
| `sign_message` | 对消息进行签名 | `message`、`network?` |
| `sign_typed_data` | 对 EIP-712 结构化数据签名 | `typedData`、`network?` |
| `sign_transaction` | 对原始交易签名 | `transaction`、`network?` |
| `get_balance` | 查询 TRX 余额 | `address`、`network?` |

所有工具均支持可选的 `network` 参数（`mainnet` / `nile` / `shasta`），默认使用 `mainnet`。

## MCP 资源

| URI | 说明 |
|---|---|
| `wallet://networks` | 可用网络及其配置信息 |
| `wallet://config` | 当前签名器配置 |

## MCP 提示词

| 提示词 | 说明 |
|---|---|
| `send-trx` | 发送 TRX 的引导式工作流 |
| `check-balance` | 查询余额的引导式工作流 |
| `send-token` | 发送 TRC20 代币的引导式工作流 |

## 工作原理

1. AI 代理调用 MCP 工具（如 `send_trx`）
2. 服务器将请求转发给 `tronlink-signer`，后者启动本地 HTTP 服务并打开浏览器授权页面
3. 授权页面通过 **TIP-6963** 协议发现 TronLink 钱包
4. 自动解锁钱包，并在需要时切换网络
5. 用户在浏览器中查看并确认请求
6. TronLink 完成交易签名，私钥始终留在钱包中
7. 结果返回给 AI 代理

## 环境变量

| 变量名 | 说明 | 默认值 |
|---|---|---|
| `TRON_NETWORK` | 默认网络（mainnet / nile / shasta） | `mainnet` |
| `TRON_HTTP_PORT` | 本地 HTTP 服务端口 | `3386` |
| `TRON_API_KEY` | TronGrid API Key（可选） | — |


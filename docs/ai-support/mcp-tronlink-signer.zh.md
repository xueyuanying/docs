# mcp-tronlink-signer

**GitHub**: https://github.com/TronLink/mcp-tronlink-signer

将 [tronlink-signer](https://github.com/TronLink/mcp-tronlink-signer/tree/main/packages/tronlink-signer) 封装为 MCP 工具的服务器，供 Claude 及其他 AI 客户端使用。通过 TronLink 浏览器钱包对 TRON 交易进行签名，需用户在浏览器中授权确认 — 私钥始终留在钱包中，不会对外暴露。

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
| ---- | ---- | ---- |
| `connect_wallet` | 连接 TronLink 钱包 | `network?` |
| `send_trx` | 向指定地址发送 TRX | `to`、`amount`、`network?` |
| `send_trc20` | 发送 TRC20 代币 | `contractAddress`、`to`、`amount`、`decimals?`、`network?` |
| `sign_message` | 对消息进行签名 | `message`、`network?` |
| `sign_typed_data` | 对 EIP-712 结构化数据签名 | `typedData`、`network?` |
| `sign_transaction` | 对原始交易签名（可选广播） | `transaction`、`broadcast?`、`network?` |
| `get_balance` | 查询 TRX 余额 | `address`、`network?` |

所有工具均支持可选的 `network` 参数（`mainnet` / `nile` / `shasta`），默认使用 `mainnet`。

## MCP 资源

| URI | 说明 |
| --- | ---- |
| `wallet://networks` | 可用网络及其配置信息 |
| `wallet://config` | 当前签名器配置 |

## MCP 提示词

| 提示词 | 说明 |
| ------ | ---- |
| `send-trx` | 发送 TRX 的引导式工作流 |
| `check-balance` | 查询余额的引导式工作流 |
| `send-token` | 发送 TRC20 代币的引导式工作流 |

## 工作原理

1. AI 智能体调用 MCP 工具（如 `send_trx`）— CLI 中会显示签名提示
2. 服务器将请求委托给 `tronlink-signer`，后者打开**单个浏览器标签页**用于授权（如已打开则复用现有标签）
3. 授权页面通过 **TIP-6963** 协议发现 TronLink 钱包
4. 自动解锁钱包，并在需要时切换网络
5. 如果钱包已连接，`connect_wallet` 会自动完成
6. 交易详情会被解析为人类可读的格式（TRX 转账、TRC20、TRC721 NFT、质押、委托、投票等）
7. 用户在浏览器中查看并批准
8. TronLink 完成交易签名 — 私钥始终留在钱包中
9. 结果返回给 AI 智能体 — 页面保持打开以便处理下一次操作

## 取消

所有签名工具均支持 MCP 取消机制。如果 AI 客户端取消了一个待处理的工具调用（例如用户在 Claude Code 中按下 Ctrl+C），进行中的请求会被自动中止，对于已被取消的请求不会打开浏览器授权页面。

## 交易确认

当调用 `sign_transaction` 且 `broadcast: true` 时，服务器会在广播后自动轮询链上确认状态，并返回执行结果（`success` 或 `pending`）。如果交易在链上失败（如 `OUT_OF_ENERGY`、Solidity revert），错误信息会连同解码后的原因一并返回给 AI 智能体。

## 环境变量

| 变量名 | 说明 | 默认值 |
| ------ | ---- | ------ |
| `TRON_NETWORK` | 默认网络（mainnet / nile / shasta） | `mainnet` |
| `TRON_HTTP_PORT` | 本地 HTTP 服务端口 | `3386` |
| `TRON_API_KEY` | TronGrid API Key（可选） | - |

# tronlink-signer

**GitHub**: https://github.com/TronLink/mcp-tronlink-signer

通过 TronLink 浏览器钱包签署 TRON 交易的独立 SDK。私钥始终留在 TronLink 中，所有签名操作均通过本地授权页面在浏览器内完成。

## 安装

```bash
npm install tronlink-signer
```

## 快速开始

```ts
import { TronSigner } from "tronlink-signer";

const signer = new TronSigner();
await signer.start();

const { address } = await signer.connectWallet();
const { txId } = await signer.sendTrx("TXxx...", 1);
const { balance } = await signer.getBalance("TXxx...");

await signer.stop();
```

## API

### `new TronSigner(config?: Partial<AppConfig>)`

创建一个新的签名实例。若未传入配置，则通过 `loadConfig()` 从环境变量中读取。

### `signer.start(): Promise<void>`

启动用于浏览器授权的本地 HTTP 服务。

### `signer.stop(): Promise<void>`

停止服务并清除所有待处理请求。

### `signer.getConfig(): AppConfig`

返回当前配置信息。

### `signer.connectWallet(network?: TronNetwork): Promise<{ address: string }>`

打开浏览器连接 TronLink，并获取钱包地址。

### `signer.sendTrx(to, amount, network?): Promise<{ txId: string }>`

向指定地址发送 TRX，会打开浏览器授权页面供用户确认。

| 参数 | 类型 | 说明 |
|---|---|---|
| `to` | `string` | 接收方 TRON 地址（base58 格式） |
| `amount` | `number` | 发送的 TRX 数量 |
| `network` | `TronNetwork` | 可选，网络覆盖参数 |

### `signer.sendTrc20(contractAddress, to, amount, decimals?, network?): Promise<{ txId: string }>`

发送 TRC20 代币，会打开浏览器授权页面。

| 参数 | 类型 | 说明 |
|---|---|---|
| `contractAddress` | `string` | TRC20 代币合约地址 |
| `to` | `string` | 接收方 TRON 地址（base58 格式） |
| `amount` | `string` | 最小单位的金额 |
| `decimals` | `number` | 代币精度（默认：6） |
| `network` | `TronNetwork` | 可选，网络覆盖参数 |

### `signer.signMessage(message, network?): Promise<{ signature: string }>`

对纯文本消息进行签名。

### `signer.signTypedData(typedData, network?): Promise<{ signature: string }>`

对 EIP-712 结构化数据进行签名。

```ts
const { signature } = await signer.signTypedData({
  domain: { name: "MyDApp", version: "1", chainId: 728126428 },
  types: {
    Greeting: [{ name: "contents", type: "string" }],
  },
  primaryType: "Greeting",
  message: { contents: "Hello Tron!" },
});
```

### `signer.signTransaction(transaction, network?): Promise<{ signedTransaction: Record<string, unknown> }>`

对原始交易进行签名，不广播到链上。

### `signer.getBalance(address, network?): Promise<{ balance: string; balanceSun: number }>`

查询指定地址的 TRX 余额，无需浏览器授权。

## 工作原理

1. 你的代码调用签名方法（如 `signMessage`）
2. 本地 HTTP 服务在端口 `3386` 启动，浏览器自动打开授权页面
3. 授权页面通过 **TIP-6963** 协议发现钱包（回退到 `window.tron`）
4. 自动解锁钱包，并在需要时切换网络
5. 用户查看请求后点击 **批准** 或 **拒绝**
6. TronLink 在浏览器中完成签名
7. 结果返回给你的代码

本地服务仅绑定到 `127.0.0.1`。若端口 `3386` 已被占用，会自动递增端口号。请求超时时间为 5 分钟。

## 网络

所有签名方法均支持可选的 `network` 参数：

| 网络 | 完整节点地址 | 区块链浏览器 |
|---|---|---|
| `mainnet`（默认） | `https://api.trongrid.io` | `https://tronscan.org` |
| `nile` | `https://nile.trongrid.io` | `https://nile.tronscan.org` |
| `shasta` | `https://api.shasta.trongrid.io` | `https://shasta.tronscan.org` |

## 环境变量

| 变量名 | 说明 | 默认值 |
|---|---|---|
| `TRON_NETWORK` | 默认网络 | `mainnet` |
| `TRON_HTTP_PORT` | 本地 HTTP 服务端口 | `3386` |
| `TRON_API_KEY` | TronGrid API Key（可选） | — |

## 类型定义

```ts
type TronNetwork = "mainnet" | "nile" | "shasta";

interface AppConfig {
  network: TronNetwork;
  httpPort: number;
  apiKey?: string;
}

interface NetworkConfig {
  name: string;
  fullHost: string;
  explorerUrl: string;
}
```

## 导出

```ts
// 类
export { TronSigner } from "./tron-signer.js";

// 配置
export { NETWORKS, DEFAULT_HTTP_PORT, REQUEST_TIMEOUT_MS, loadConfig } from "./config.js";

// 类型
export type {
  TronNetwork, NetworkConfig, AppConfig,
  PendingRequestType, PendingRequest,
  ConnectData, SendTrxData, SendTrc20Data,
  SignMessageData, SignTypedDataData, SignTransactionData,
} from "./types.js";
```

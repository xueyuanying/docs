# TronLink CLI

**GitHub**: [https://github.com/TronLink/tronlink-cli](https://github.com/TronLink/tronlink-cli)

通过 TronLink 钱包签名实现 TRON 区块链操作的命令行工具。

所有交易均在本地构建，并通过 TronLink 浏览器扩展进行签名 — 私钥始终不会离开 TronLink。

## 环境要求

- Node.js >= 20
- 已安装 TronLink 浏览器扩展
- 浏览器处于运行状态（写操作会打开 TronLink 进行签名）

## 安装

```bash
# 通过 npm 安装
npm install -g @tronlink/tronlink-cli

# 本地开发
npm install
npm run build
npm link
```

安装完成后，可在全局使用 `tronlink` 命令。

## 全局选项

| 选项 | 默认值 | 说明 |
| ---- | ------ | ---- |
| `--local-broadcast` | 关闭 | 由 CLI 本地广播，而不是让签名器广播 |
| `--json` | 关闭 | 以 JSON 格式输出，便于脚本/AI 智能体处理 |
| `--port <n>` | 3386 | TronLink Signer HTTP 服务端口 |
| `--api-key <key>` | - | TronGrid API Key（也可设置 `TRON_API_KEY` 环境变量） |
| `--timeout <ms>` | 300000 | 签名/连接超时时间（毫秒） |

所有选项名称均**不区分大小写**（例如 `--toAddress`、`--TOADDRESS`、`--toaddress` 完全等价）。

## 命令

### 会话（serve 守护进程）

持久化的签名守护进程会在多个命令之间复用，因此浏览器标签页保持打开，每个会话只需确认一次网络/账户授权。

```bash
# 前台启动（立即连接钱包）
tronlink serve [--network nile]

# 停止运行中的守护进程
tronlink serve stop

# 显式连接并验证钱包
tronlink connect [--network nile]

# 查看或切换钱包当前网络
tronlink network
tronlink network --network nile
```

无需手动启动 `serve` — 其他命令首次使用时会自动派生一个后台守护进程（日志位于 `~/.tronlink-cli/daemon.log`）。当浏览器标签页关闭时，守护进程会自动关闭。

### 查询（读操作）

读命令支持两种模式：

- **传入 `--address`**：直接通过 TronGrid 查询，无需连接钱包。如未传 `--network`，默认查询主网。
- **不传 `--address`**：连接钱包以从 TronLink 实时获取地址和网络。

```bash
# 查询所有代币余额
tronlink balance --address <address> [--network mainnet]
tronlink balance                                          # 连接钱包

# 查询指定 TRC20 代币余额（自动检测精度）
tronlink balance --address <address> --token <contract> [--decimals 6] [--network mainnet]

# 查询指定 TRC10 代币余额（自动检测精度）
tronlink balance --address <address> --tokenId <id> [--decimals 6] [--network mainnet]

# 查询能量与带宽
tronlink resource --address <address> [--network nile]
tronlink resource                                         # 连接钱包
```

**主网查询代币**：TRX, USDT, USDD, USDC, SUN, JST, BTT, WIN, WTRX
**Nile 查询代币**：TRX, USDT
**Shasta 查询代币**：TRX

### 转账

```bash
# TRX（金额单位为 TRX，而非 sun）
tronlink transfer --type trx --toAddress <to> --amount <amount> [--network nile]

# TRC10（自动检测精度，或通过 --decimals 指定）
tronlink transfer --type trc10 --tokenId <id> --toAddress <to> --amount <amount> [--decimals 6] [--network nile]

# TRC20（自动检测精度；可选 --fee-limit 单位为 TRX，默认 100）
tronlink transfer --type trc20 --contract <contract> --toAddress <to> --amount <amount> [--decimals 6] [--fee-limit 150] [--network nile]

# TRC721 NFT（可选 --fee-limit 单位为 TRX，默认 100）
tronlink transfer --type trc721 --contract <contract> --toAddress <to> --tokenId <id> [--fee-limit 150] [--network nile]
```

示例：

```bash
tronlink transfer --type trx --toAddress TYqx5gm3p3wLDE9Bv8TBJAbK4ELNbSLfJV --amount 100
tronlink transfer --type trc20 --contract TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t --toAddress TYqx5gm3p3wLDE9Bv8TBJAbK4ELNbSLfJV --amount 50
tronlink transfer --type trc721 --contract TContractAddr --toAddress TRecipient --tokenId 12345
```

各类型的参数校验：

| 类型 | 必需参数 | 不允许的参数 |
| ---- | -------- | ------------ |
| trx | `--toAddress`, `--amount` | `--tokenId`, `--contract`, `--decimals`, `--fee-limit` |
| trc10 | `--toAddress`, `--amount`, `--tokenId` | `--contract`, `--fee-limit` |
| trc20 | `--toAddress`, `--amount`, `--contract` | `--tokenId` |
| trc721 | `--toAddress`, `--contract`, `--tokenId` | `--amount`, `--decimals` |

### 质押（Stake 2.0）

```bash
# 质押 TRX 以获取能量或带宽
tronlink stake --amount <amount> --resource <energy|bandwidth> [--network nile]

# 解锁已质押的 TRX
tronlink unstake --amount <amount> --resource <energy|bandwidth> [--network nile]

# 提取已解冻的 TRX（在 14 天解锁期之后）
tronlink withdraw [--network nile]
```

### 资源代理

```bash
# 代理能量或带宽（锁定期单位为天，支持小数如 1.5）
tronlink delegate --toAddress <to> --amount <amount> --resource <energy|bandwidth> [--lock-period <days>] [--network nile]

# 回收已代理的资源
tronlink reclaim --fromAddress <from> --amount <amount> --resource <energy|bandwidth> [--network nile]
```

### 投票

```bash
# 为超级代表投票（格式：address:count，支持多个）
tronlink vote --votes <address:count...> [--network nile]

# 领取投票奖励
tronlink reward [--network nile]
```

示例：

```bash
tronlink vote --votes TXxx:5 TYyy:3 TZzz:2
tronlink reward
```

## 交易签名

写操作（转账、质押、代理、投票等）需要在浏览器中通过 TronLink 进行授权：

1. CLI 在本地构建交易并显示预览
2. 浏览器打开 TronLink Signer 授权页面
3. 用户审核后点击 **批准** 或 **拒绝**
4. 结果返回 CLI

借助共享的 `serve` 守护进程，多个命令会复用同一个浏览器窗口 — 每个会话只需要一个浏览器标签页。关闭该标签页会停止守护进程。

支持多个命令并发执行，浏览器 UI 中每个待处理请求会以独立标签呈现，可按任意顺序进行授权。

取消某个命令（Ctrl+C）只会取消该笔交易，其他排队中的交易不受影响。

## 交易预览

所有写操作在签名前都会显示预览：

```text
Transaction Preview
┌───────────┬────────────────────────────────────────┐
│ Action    │ Transfer TRX                           │
│ Network   │ nile                                   │
│ From      │ TXxx...                                │
│ To        │ TYyy...                                │
│ Amount    │ 100 TRX                                │
│ Broadcast │ Signer                                 │
└───────────┴────────────────────────────────────────┘
Awaiting TronLink approval...
```

## 广播

默认情况下，签名后的交易由签名器（TronLink）广播。使用 `--local-broadcast` 可让 CLI 通过自身的 TronWeb 实例进行本地广播：

```bash
# 默认：签名器签名后广播
tronlink transfer --type trx --toAddress TYqx5gm3p3wLDE9Bv8TBJAbK4ELNbSLfJV --amount 100

# CLI 在本地广播
tronlink transfer --type trx --toAddress TYqx5gm3p3wLDE9Bv8TBJAbK4ELNbSLfJV --amount 100 --local-broadcast
```

## 输入校验

所有输入在连接 TronLink 前会进行校验：

- **金额**：仅支持非负数字，不允许科学计数法或多个小数点
- **TRX 金额**：必须 > 0，转换为 sun 后需在安全整数范围内
- **TRC20/TRC10 金额**：小数位数不能超过该代币的精度。如未传入 `--decimals`，会从链上自动检测
- **地址**：必须为有效的 TRON 地址格式（通过 `TronWeb.isAddress()` 校验）
- **合约存在性**：在查询精度前会先在指定网络上验证
- **投票数量**：正整数，且不允许重复的 SR 地址
- **网络**：必须为 `mainnet`、`nile` 或 `shasta`
- **端口**：1 到 65535 之间的整数
- **精度**：非负整数（0-77）
- **手续费上限**：以 TRX 为单位的正数（仅 TRC20/TRC721）
- **转账类型校验**：缺少必需参数或包含不适用的额外参数将被明确报错拒绝

无效输入会在任何钱包交互发生之前立即被拒绝并提示明确错误。

## 输出格式

**表格（默认）**：人类可读的表格输出。

**JSON（`--json`）**：面向脚本和 AI 智能体的机器可读输出。

## 支持的网络

| 网络 | API 节点 | 区块链浏览器 |
| ---- | -------- | ------------ |
| mainnet | `https://api.trongrid.io` | `https://tronscan.org` |
| nile | `https://nile.trongrid.io` | `https://nile.tronscan.org` |
| shasta | `https://api.shasta.trongrid.io` | `https://shasta.tronscan.org` |

## 工作原理

1. CLI 解析命令并校验所有输入
2. 对于带 `--address` 的读操作：直接通过 TronGrid 查询
3. 对于写操作或不带 `--address` 的读操作：通过 IPC 连接 `serve` 守护进程（如未运行则自动派生一个），守护进程与 TronLink 浏览器扩展通信
4. 使用本地 TronWeb `transactionBuilder` 构建未签名交易
5. 将交易发送给 TronLink 进行签名（浏览器授权页面）
6. 广播：默认由签名器广播（返回 txId）；使用 `--local-broadcast` 时由 CLI 在本地广播并验证结果
7. 输出结果

## AI / 智能体使用

TronLink CLI 通过 `--json` 输出支持 AI 智能体集成。所有命令都会返回结构化 JSON，便于解析。

### 前置条件

- 已全局安装 `tronlink`（`npm i -g @tronlink/tronlink-cli`）
- 已安装并解锁 TronLink 浏览器扩展
- 浏览器处于运行状态（写操作会打开 TronLink 进行签名）

### 使用规则

1. 始终追加 `--json` 以获取机器可读输出
2. 写操作（转账、质押、投票等）会打开浏览器供用户签名 — 需等待命令返回（默认超时：5 分钟）
3. 带 `--address` 的读操作无需连接钱包，查询更快
4. 通过 `--network` 指定网络，否则默认使用钱包当前网络（写操作）或主网（带地址的读操作）
5. 取消命令（Ctrl+C）只会取消该笔交易，不会终止签名会话

### AI 命令参考

#### 查询（无需签名）

```bash
# 查询所有代币余额
tronlink balance --address <address> --network mainnet --json

# 查询指定 TRC20 代币余额
tronlink balance --address <address> --token <contract> --network mainnet --json

# 查询指定 TRC10 代币余额
tronlink balance --address <address> --tokenId <id> --network mainnet --json

# 查询能量与带宽
tronlink resource --address <address> --network mainnet --json
```

#### 转账（需要签名）

```bash
# TRX
tronlink transfer --type trx --toAddress <to> --amount <amount> --json

# TRC20（自动检测精度）
tronlink transfer --type trc20 --contract <contract> --toAddress <to> --amount <amount> --json

# TRC10（自动检测精度）
tronlink transfer --type trc10 --tokenId <id> --toAddress <to> --amount <amount> --json

# TRC721 NFT
tronlink transfer --type trc721 --contract <contract> --toAddress <to> --tokenId <id> --json
```

#### 质押

```bash
tronlink stake --amount <amount> --resource energy --json
tronlink unstake --amount <amount> --resource energy --json
tronlink withdraw --json
```

#### 资源代理

```bash
tronlink delegate --toAddress <to> --amount <amount> --resource energy --json
tronlink reclaim --fromAddress <from> --amount <amount> --resource energy --json
```

#### 投票

```bash
tronlink vote --votes <addr:count...> --json
tronlink reward --json
```

#### 会话 / 网络

```bash
tronlink connect --json
tronlink network --json
tronlink network --network nile --json
tronlink serve stop
```

### 常用代币合约

| 代币 | 网络 | 合约地址 |
| ---- | ---- | -------- |
| USDT | mainnet | TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t |
| USDD | mainnet | TXDk8mbtRbXeYuMNS83CfKPaYYT8XWv9Hz |
| USDC | mainnet | TEkxiTehnzSmSe2XqrBj4w32RUN966rdz8 |

### 示例：AI 转账流程

```bash
# 1. 先查询余额
tronlink balance --address TNPeeaaFB7K9cmo4uQpcU32zGK8G1NYqeL --network mainnet --json

# 2. 发送 10 TRX（会打开浏览器签名，需等待返回）
tronlink transfer --type trx --toAddress TRecipientAddress --amount 10 --network mainnet --json

# 3. 验证结果 — 输出包含 txId 和浏览器链接
# { "Status": "Success", "TxID": "abc...", "Explorer": "https://tronscan.org/#/transaction/abc..." }
```

### 注意事项

- 所有写命令都会阻塞，直到用户在 TronLink 浏览器弹窗中批准或拒绝
- 如用户拒绝或签名超时（5 分钟），命令会以错误退出
- 取消 CLI 命令（Ctrl+C）只会取消该笔交易 — 其他排队中的交易将继续执行
- 使用 `--timeout <ms>` 可调整签名超时时间
- 金额内部使用基于字符串的运算 — 不存在浮点精度问题

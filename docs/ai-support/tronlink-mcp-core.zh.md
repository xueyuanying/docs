# TronLink MCP Core 

## 概述

**GitHub**: [https://github.com/TronLink/tronlink-mcp-core](https://github.com/TronLink/tronlink-mcp-core)

**@tronlink/tronlink-mcp-core** 是构建 TronLink MCP（Model Context Protocol）服务器的基础框架库。它不是独立应用——使用者必须实现 `ISessionManager` 接口并注入能力模块来创建可运行的服务器。

**核心亮点：**
- 接口驱动的可插拔架构，提供 **9 个能力接口**
- **56+ 预定义工具处理器**，使用 Zod 验证 Schema
- 内置 Knowledge Store，支持跨会话学习和步骤回放
- Flow Recipe 系统，将多步骤工作流程模板化
- 标准化响应格式，25+ 错误码
- 双模支持：Playwright（UI 自动化）+ Direct API（链上操作）

---

## 架构设计

```
┌─ AI 代理 (Claude, GPT 等)
│
├─ MCP 协议 (stdio / JSON-RPC 2.0)
│
├─ MCP 服务器实例 (createMcpServer)
│  ├─ 56+ tl_* 工具处理器
│  ├─ Knowledge Store（跨会话持久化）
│  ├─ Flow Registry（流程管理）
│  └─ Discovery 工具
│
├─ ISessionManager 接口（使用者实现）
│  ├─ 会话生命周期
│  ├─ 页面/标签管理
│  ├─ 能力注入（9 个接口）
│  └─ 环境模式 (e2e / prod)
│
├─ 能力系统（9 个可插拔接口）
│  ├─ BuildCapability
│  ├─ FixtureCapability
│  ├─ ChainCapability
│  ├─ ContractSeedingCapability
│  ├─ StateSnapshotCapability
│  ├─ MockServerCapability
│  ├─ OnChainCapability
│  ├─ MultiSigCapability
│  └─ GasFreeCapability
│
└─ 可选: Playwright（浏览器自动化）
```

**设计原则：**
1. **接口驱动** — 所有主要组件使用接口实现可扩展性
2. **组合优于继承** — 能力通过注入，而非继承
3. **单例模式** — SessionManager、KnowledgeStore、FlowRegistry 使用全局持有者
4. **预检查** — 链上工具在执行前验证
5. **数据脱敏** — Knowledge Store 自动屏蔽敏感字段

---

## 与 mcp-server-tronlink 的关系

| 维度 | tronlink-mcp-core | mcp-server-tronlink |
|------|-------------------|---------------------|
| 类型 | 核心库（框架） | 独立 MCP 服务器 |
| 角色 | 定义接口、工具、协议 | 具体实现 |
| 使用方式 | 通过 npm 导入并扩展 | 直接 CLI 调用 |
| 可扩展性 | 9 个可插拔能力接口 | 预配置能力 |
| 依赖关系 | 无（它本身就是依赖） | 依赖 @tronlink/tronlink-mcp-core |

**mcp-server-tronlink 是 tronlink-mcp-core 的使用者。** 核心库定义了工具"是什么"；服务器提供了工具"怎么工作"。

---

## ISessionManager 接口

使用者必须实现的关键接口（25+ 方法）：

### 会话生命周期
```typescript
hasActiveSession(): boolean
getSessionId(): string
getSessionState(): SessionState
getSessionMetadata(): SessionMetadata
launch(input: LaunchInput): Promise<LaunchResult>
cleanup(): Promise<void>
```

### 页面管理
```typescript
getPage(): Page
setActivePage(page: Page): void
getTrackedPages(): TrackedPage[]
classifyPageRole(page: Page): PageRole
getContext(): ContextInfo
```

### 扩展状态
```typescript
getExtensionState(): Promise<ExtensionState>
```

### 无障碍引用
```typescript
setRefMap(map: Map<string, any>): void
getRefMap(): Map<string, any>
clearRefMap(): void
resolveA11yRef(ref: string): any
```

### 导航
```typescript
navigateToHome(): Promise<void>
navigateToSettings(): Promise<void>
navigateToUrl(url: string): Promise<void>
navigateToNotification(): Promise<void>
waitForNotificationPage(timeoutMs?: number): Promise<Page>
```

### 截图
```typescript
screenshot(options?: ScreenshotOptions): Promise<ScreenshotResult>
```

### 能力获取器（9 个，均可选）
```typescript
getBuildCapability(): BuildCapability | undefined
getFixtureCapability(): FixtureCapability | undefined
getChainCapability(): ChainCapability | undefined
getContractSeedingCapability(): ContractSeedingCapability | undefined
getStateSnapshotCapability(): StateSnapshotCapability | undefined
getMockServerCapability(): MockServerCapability | undefined
getOnChainCapability(): OnChainCapability | undefined
getMultiSigCapability(): MultiSigCapability | undefined
getGasFreeCapability(): GasFreeCapability | undefined
```

### 环境
```typescript
getEnvironmentMode(): 'e2e' | 'prod'
setContext(context: string, options?: any): Promise<void>
getContextInfo(): ContextInfo
```

---

## 9 个能力接口

每个能力都是可选的，可独立注入：

### 1. BuildCapability
从源码构建 TronLink 扩展。

### 2. FixtureCapability
管理钱包状态 JSON（default、onboarding、自定义预设）。

### 3. ChainCapability
控制本地 TRON 节点（tron-quickstart 等）。

### 4. ContractSeedingCapability
部署智能合约（TRC20/721/1155/10/multisig/staking/energy_rental）。

### 5. StateSnapshotCapability
从 UI 提取钱包状态（界面、地址、余额、能量、带宽）。

### 6. MockServerCapability
用于隔离测试的 Mock API 服务器。

### 7. OnChainCapability
通过 TronGrid REST API 的直接链上操作（14 个方法）：
- 查询：`getAddress`、`getAccount`、`getTokens`、`getTransaction`、`getHistory`、`getStakingInfo`
- 交易：`send`、`stake`、`resource`、`swap`、`swapV3`
- 多签：`setupMultisig`、`createMultisigTx`、`signMultisigTx`

### 8. MultiSigCapability
多签服务集成（REST + WebSocket，5 个方法）。

### 9. GasFreeCapability
零 Gas TRC20 转账服务（3 个方法）。

---

## 56+ 工具定义

所有工具使用 `tl_` 前缀，分为 13 个类别：

### 1. 会话管理（2 个）
| 工具 | 说明 |
|------|------|
| `tl_launch` | 启动带扩展的浏览器 |
| `tl_cleanup` | 关闭浏览器和服务 |

### 2. 状态与发现（4 个）
| 工具 | 说明 |
|------|------|
| `tl_get_state` | 获取钱包状态 |
| `tl_describe_screen` | 包含状态 + testIds + a11y + 截图的界面描述 |
| `tl_list_testids` | 列出 data-testid 属性 |
| `tl_accessibility_snapshot` | 获取带引用的无障碍树 (e1, e2...) |

### 3. 导航（4 个）
| 工具 | 说明 |
|------|------|
| `tl_navigate` | 导航到 TronLink 界面或 URL |
| `tl_switch_to_tab` | 按角色/URL 切换标签页 |
| `tl_close_tab` | 关闭标签页 |
| `tl_wait_for_notification` | 等待确认弹窗 |

### 4. UI 交互（6 个）
| 工具 | 说明 |
|------|------|
| `tl_click` | 点击元素（a11yRef、testId、selector） |
| `tl_type` | 在输入框中输入文本 |
| `tl_wait_for` | 等待元素状态 |
| `tl_scroll` | 滚动页面/元素 |
| `tl_keyboard` | 发送键盘事件 |
| `tl_evaluate` | 执行 JavaScript |

### 5. 截图与剪贴板（2 个）
| 工具 | 说明 |
|------|------|
| `tl_screenshot` | 捕获屏幕 |
| `tl_clipboard` | 读写剪贴板 |

### 6. 合约部署 — 仅 e2e（4 个）
| 工具 | 说明 |
|------|------|
| `tl_seed_contract` | 部署单个合约 |
| `tl_seed_contracts` | 批量部署合约 |
| `tl_get_contract_address` | 查询合约地址 |
| `tl_list_contracts` | 列出已部署合约 |

### 7. 上下文管理（2 个）
| 工具 | 说明 |
|------|------|
| `tl_set_context` | 切换 e2e/prod 模式 |
| `tl_get_context` | 获取上下文信息 |

### 8. Knowledge Store（4 个）
| 工具 | 说明 |
|------|------|
| `tl_knowledge_last` | 获取最近 N 个步骤 |
| `tl_knowledge_search` | 搜索历史 |
| `tl_knowledge_summarize` | 生成配方 |
| `tl_knowledge_sessions` | 列出会话 |

### 9. Flow Recipes（1 个）
| 工具 | 说明 |
|------|------|
| `tl_list_flows` | 列出/获取流程配方 |

### 10. 批量执行（1 个）
| 工具 | 说明 |
|------|------|
| `tl_run_steps` | 顺序执行多个步骤 |

### 11. 链上操作（14 个）
| 工具 | 说明 |
|------|------|
| `tl_chain_get_address` | 从私钥获取地址 |
| `tl_chain_get_account` | 查询账户详情 |
| `tl_chain_get_tokens` | 查询 TRC10/TRC20 余额 |
| `tl_chain_send` | 发送 TRX/TRC10/TRC20 |
| `tl_chain_get_tx` | 获取交易详情 |
| `tl_chain_get_history` | 查询交易历史 |
| `tl_chain_stake` | 冻结/解冻 TRX (Stake 2.0) |
| `tl_chain_get_staking` | 查询质押信息 |
| `tl_chain_resource` | 代理/取消代理资源 |
| `tl_chain_swap` | SunSwap V2 兑换 |
| `tl_chain_swap_v3` | SunSwap V3 兑换 |
| `tl_chain_setup_multisig` | 配置多签权限 |
| `tl_chain_create_multisig_tx` | 创建未签名多签交易 |
| `tl_chain_sign_multisig_tx` | 签署多签交易 |

### 12. 多签管理（5 个）
| 工具 | 说明 |
|------|------|
| `tl_multisig_query_auth` | 查询多签权限 |
| `tl_multisig_submit_tx` | 提交签名交易 |
| `tl_multisig_list_tx` | 列出多签交易 |
| `tl_multisig_connect_ws` | 连接 WebSocket |
| `tl_multisig_disconnect_ws` | 断开 WebSocket |

### 13. GasFree（3 个）
| 工具 | 说明 |
|------|------|
| `tl_gasfree_get_account` | 查询资格与配额 |
| `tl_gasfree_get_transactions` | 查询交易历史 |
| `tl_gasfree_send` | 零 Gas 发送 |

---

## 标准化响应格式

所有工具返回一致的结构：

```typescript
// 成功
{
  ok: true,
  result: { /* 工具特定数据 */ },
  meta: {
    timestamp: "2026-03-09T10:15:23.456Z",
    sessionId: "tl-1741504523",
    durationMs: 234
  }
}

// 错误
{
  ok: false,
  error: {
    code: "TL_CLICK_FAILED",       // 25+ 错误码之一
    message: "Element not found",
    details: { /* 可选 */ }
  },
  meta: { timestamp, sessionId, durationMs }
}
```

---

## Knowledge Store

跨会话学习和步骤回放系统。

### 存储结构
```
test-artifacts/llm-knowledge/
├── tl-1741504523/
│   ├── session.json
│   └── steps/
│       ├── 2026-03-09T10-15-23-456Z-tl_click.json
│       └── ...
└── tl-1741504600/
    └── ...
```

### 特性
- **自动记录：** 每次工具调用都被记录（时间戳、界面、目标、结果）
- **敏感数据脱敏：** 密码、助记词、私钥、种子自动屏蔽
- **搜索：** 按工具名、界面、testId、无障碍名称查询
- **摘要生成：** 从会话历史生成可复用的"配方"
- **会话管理：** 带元数据的会话列表

---

## Flow Recipe 系统

将常见多步骤工作流程模板化，支持参数替换。

### FlowRecipe 结构
```typescript
{
  id: "transfer_trx",
  name: "发送 TRX",
  description: "向另一个地址转账 TRX",
  context: "both",                      // "playwright" | "api" | "both"
  preconditions: ["钱包已解锁"],
  params: [
    { name: "recipient", description: "目标 TRON 地址", required: true },
    { name: "amount", description: "TRX 金额", required: true }
  ],
  steps: [
    { tool: "navigate", input: { target: "send" } },
    { tool: "type", input: { testId: "address-input", text: "{{recipient}}" } },
    { tool: "type", input: { testId: "amount-input", text: "{{amount}}" } },
    { tool: "click", input: { a11yRef: "e5" } }
  ],
  tags: ["transfer", "basic"]
}
```

---

## 元素定位（3 种方式）

```typescript
// 1. 无障碍引用（推荐）
tl_click({ a11yRef: "e5" })

// 2. data-testid
tl_click({ testId: "send-button" })

// 3. CSS 选择器
tl_click({ selector: ".confirm-btn" })
```

无障碍引用来自 `tl_accessibility_snapshot` 输出——最可靠的定位方式。

---

## 安装与使用

### 安装
```bash
npm install @tronlink/tronlink-mcp-core
```

### 基础使用
```typescript
import {
  createMcpServer,
  setSessionManager,
  type ISessionManager,
} from '@tronlink/tronlink-mcp-core';

// 1. 实现 ISessionManager
class MySessionManager implements ISessionManager {
  // ... 实现全部 25+ 方法
}

// 2. 注册
setSessionManager(new MySessionManager());

// 3. 创建并启动服务器
const server = createMcpServer({
  name: 'My TronLink Server',
  version: '1.0.0',
});

await server.start();
```

---

## 项目结构

```
tronlink-mcp-core/
├── src/
│   ├── index.ts                           # 公共 API 导出
│   ├── mcp-server/
│   │   ├── server.ts                      # createMcpServer() 工厂函数
│   │   ├── session-manager.ts             # ISessionManager 接口
│   │   ├── knowledge-store.ts             # 持久化步骤记录
│   │   ├── discovery.ts                   # 页面检查工具
│   │   ├── schemas.ts                     # Zod 验证 Schema
│   │   ├── constants.ts                   # 超时、限制、URL、界面常量
│   │   ├── tools/                         # 56+ 工具处理器
│   │   ├── types/                         # 类型定义
│   │   └── utils/                         # 工具函数
│   ├── capabilities/
│   │   ├── types.ts                       # 9 个能力接口
│   │   └── context.ts                     # 环境配置类型
│   ├── flows/
│   │   ├── types.ts                       # FlowRecipe 类型
│   │   └── registry.ts                    # FlowRegistry 类
│   ├── launcher/
│   │   ├── extension-id-resolver.ts       # 扩展 ID 提取
│   │   └── extension-readiness.ts         # 扩展加载检测
│   └── utils/
│       └── index.ts                       # 工具函数
├── dist/                                  # 编译输出 + .d.ts
├── package.json
├── tsconfig.json
└── README.md
```

---

## 依赖项

| 包名 | 版本 | 用途 |
|------|------|------|
| `@modelcontextprotocol/sdk` | ^1.12.0 | MCP 协议实现 |
| `zod` | ^3.23.0 | 输入验证 Schema |

**可选对等依赖：**

| 包名 | 版本 | 用途 |
|------|------|------|
| `playwright` | ^1.49.0 | 浏览器自动化（仅 Playwright 模式） |
| `@playwright/test` | ^1.49.0 | 测试工具 |

---

## 构建与开发

```bash
npm run build      # 编译 TypeScript 到 dist/
npm run dev        # 监视模式
npm run test       # 运行 Vitest 测试
npm run lint       # ESLint
npm run clean      # 删除 dist/
```

---

## 关键设计模式

1. **全局单例** — `setSessionManager()` / `getSessionManager()`，`setKnowledgeStore()` / `getKnowledgeStore()`，`FlowRegistry.getInstance()`
2. **组合优于继承** — 能力通过 getter 方法注入，而非类层次结构
3. **接口隔离** — 每个能力都有专注的最小接口
4. **预检查** — 链上工具在发送交易前验证余额、权限、配额
5. **数据脱敏** — Knowledge Store 自动屏蔽 password、mnemonic、private_key、seed 字段
6. **标准化响应** — 所有 56+ 工具使用一致的 `{ ok, result/error, meta }` 结构
7. **模板系统** — Flow Recipes 使用 `{{param}}` 占位符实现可复用工作流

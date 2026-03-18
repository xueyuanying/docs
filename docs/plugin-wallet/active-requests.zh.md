# 主动请求TronLink插件功能

### 连接网站 TIP-1102

#### 简介
TronLink可用于管理钱包私钥，dapp在进行一些需签名的操作前，需要连接TronLink，并通过TronLink获取用户签名授权。此协议在于显式地告知用户dapp主动连接TronLink的行为，并获取用户的授权同意。
此方法遵循以太坊EIP-1102协议。

#### 技术规范
##### 代码示例
```javascript
try {
  await tron.request({method: 'eth_requestAccounts'});
} catch (e) {}
```
##### 返回值

如果成功，返回一个数组，数组只有一个元素，为同意连接的当前TronLink账户。如：
`['TMVQGm1qAQYVdetCeGRRkTWYYrLXuHK2HC']`。

如果失败，则会返回错误码及报错信息。详见“错误码”部分。

##### 错误码
|  错误码   | 名称  | 描述 |
|  ----  | ----  | ---- |
| 4001  | 用户拒绝请求 | 用户通过点击“拒绝”按钮，或关闭弹窗，都会触发该错误码 |
| -32002  | 处于其他流程中 | 当前dapp存在其他流程，无法执行当前请求 |
| -32602  | 参数不合规 | 传入的参数不合规，或者传入了额外的参数 |
| 4200  | 不支持此方法 | 不支持此方法 |

#### 交互流程
触发`eth_requestAccounts`之后，如果TronLink处于锁屏状态，会弹出锁屏弹窗：

![image](../images/zh_plugin-wallet_lock-page.png)

解锁后，或者TronLink已经提前解锁，会打开连接确认的弹窗：

![image](../images/zh_plugin-wallet_request-accounts.png)


### 连接网站（旧版）

该连接方法已不推荐使用，之后的连接网站方法请参考[连接网站 TIP-1102](#tip-1102)
#### 简介
TronLink提供外部发起Trx转账，合约签名，授权等功能，基于安全的考虑，
需要用户在关键操作前先对发起请求的Dapp进行【连接网站】授权，在授权成功后才允许操作。
所以Dapp要先进行【连接网站】操作，等待用户允许后，方能发起需要授权的请求。

#### 技术规范
##### 代码示例
```typescript
const res = await tronWeb.request(
  {
    method: 'tron_requestAccounts',
    params: {
      websiteIcon: '<WEBSITE ICON URI>',
      websiteName: '<WEBSITE NAME>',
    } as RequestAccountParams,
  }
);
```

##### 参数
```typescript
interface RequestAccountsParams {
  websiteIcon?: string;
  websiteName?: string;
}
```
* method: tron_requestAccounts 固定的字符串
* params: RequestAccountParams类型，具体参数如下
    * websiteIcon: DApp网站的Icon的URI, 具体会展示在用户已连接网站列表中
    * websiteName: dapp网站名称


##### 返回值
类型说明
```typescript
interface ReqestAccountsResponse {
  code: 200 | 4000 | 4001,
  message: string
}
```

|  返回码   | 描述  | 返回消息 |
|  ----  | ----  | ---- |
| 无  | 钱包处于锁定状态 | 空字符串 |
| 200  | 网站此前已被用户允许连接 | The site is already in the whitelist |
| 200  | 用户同意连接 | User allowed the request. |
| 4000  | 当前请求前已经有同一个dapp发起了连接网站请求，并且弹窗仍未关闭 | Authorization requests are being processed, please do not resubmit |
| 4001  | 用户拒绝连接 | User rejected the request |

### 获取TronLink的provider TIP-6963

#### 简介
当多个钱包同时存在，会出现对`window.tron`对象的抢占行为。为了保证DApp可以获取到特定钱包的provider，所以实现TIP-6963规范
#### 技术规范
##### 代码示例
```typescript
interface TIP1193Provider {
  request: (args: RequestArguments) => Promise<unknown>;
  on(event: string, listener: (...args: any[]) => void): this;
  removeListener(event: string, listener: (...args: any[]) => void): this;
  tronWeb: TronWeb;
  [key: `is${string}`]: boolean;
}

/**
 * Represents the assets needed to display a wallet
 */
interface TIP6963ProviderInfo {
  uuid: string;
  name: string;
  icon: string;
  rdns: string;
}

interface TIP6963ProviderDetail {
  info: TIP6963ProviderInfo;
  provider: TIP1193Provider;
}

// Announce Event dispatched by a Wallet
interface TIP6963AnnounceProviderEvent extends CustomEvent {
  type: "TIP6963:announceProvider";
  detail: TIP6963ProviderDetail;
}

// The DApp listens to announced providers
window.addEventListener(
  "TIP6963:announceProvider",
  (event: TIP6963AnnounceProviderEvent) => {
    
    // Confirm if it is a Tronlink UUID
    if (event.detail.info.rdns !== 'org.tronlink.www' || event.detail.info.name !== 'TronLink') {
      console.error('it is NOT TronLink provider');
      return;
    }

    // event.detail.provider === window.tron
    const tronProvider = event.detail.provider;

    tronProvider.on('accountsChanged', (accountArray) => {
      console.log('tip-6963 accountsChanged', accountArray);
    })
  }
);

// The DApp dispatches a request event which will be heard by 
// Wallets' code that had run earlier
window.dispatchEvent(new Event("TIP6963:requestProvider"));
```
DApp按照上述代码实现后，可以精准获取到TronLink提供的provider。
TronLink 的 rdns 是 `org.tronlink.www`, name 是 `TronLink`

### 普通转账

#### 简介
dapp需要用户发起一笔trx转账

前提: dapp开发者完成[连接网站](#tip-1102)请求，用户同意连接

波场网络上发起转账需要3个步骤，

1. 构造转账交易
2. 对交易进行签名
3. 对签名后的交易进行广播

在这里，TronLink介入的是第2步签名的部分，1,3两步需要开发者使用tronWeb完成
#### 技术规范
##### 代码示例
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  const fromAddress = tronweb.defaultAddress.base58;
  const toAddress = "TDvSsdrNM5eeXNL3czpa6AxLDHZA9nwe9K";
  const tx = await tronweb.transactionBuilder.sendTrx(toAddress, 10, fromAddress); // 步骤1
  try {
    const signedTx = await tronweb.trx.sign(tx); // 步骤2
    await tronweb.trx.sendRawTransaction(signedTx); // 步骤3
  } catch (e) {}
}
```

当代码执行到`await tronweb.trx.sign(tx);`时，TronLink钱包会提示弹窗，需要用户进行确认，如下图

![image](../images/zh_plugin-wallet_sign_trx.png)

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理

如果用户在弹窗中选择【签名】，dapp可以拿到签名后的交易，继续进行广播


### 多签转账
#### 简介
此处可参考[普通转账](#_13)

#### 技术规范
##### 代码示例
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  const toAddress = "TDvSsdrNM5eeXNL3czpa6AxLDHZA9nwe9K";
  const activePermissionId = 2;
  const tx = await tronweb.transactionBuilder.sendTrx(
      toAddress, 10, 
      { permissionId: activePermissionId}
  ); // 步骤1
  try {
    const signedTx = await tronweb.trx.multiSign(tx, undefined, activePermissionId); // 步骤2
    await tronweb.trx.sendRawTransaction(signedTx); // 步骤3
  } catch (e) {}
}
```

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理

如果用户在弹窗中选择【签名】，dapp可以拿到签名后的交易，继续进行广播



### 消息签名

#### 简介
dapp需要用户对一个hex消息签名，签名后消息转发给后端进行验签，以此判断用户合法登陆

#### 前提
dapp开发者完成【连接网站】请求，用户同意连接

#### 技术规范
##### 代码示例
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  try {
    const message = "0x01EF"; // any hex string
    const signedString = await tronweb.trx.signMessageV2(message);
  } catch (e) {}
}
```
##### 参数
`tronLink.tronWeb.trx.signMessageV2`接收一个十六进制的字符串作为参数，该字符串表示当前待签名的内容

##### 返回值
如果用户在弹窗中选择签名, dapp可以得到签名后的十六进制字符串, 比如：
```
0xaa302ca153b10dff25b5f00a7e2f603c5916b8f6d78cdaf2122e24cab56ad39a79f60ff3916dde9761baaadea439b567475dde183ee3f8530b4cc76082b29c341c
```
如果报错，则会返回如下信息
```typescript
Uncaught (in promise) Invalid transaction provided
```


#### 交互流程

当代码执行到`await tronweb.trx.signMessageV2(message);`时，TronLink钱包会提示弹窗，需要用户进行确认，
如下图, 其中消息内容会以hex的方式展示

![image](../images/zh_plugin-wallet_sign_message.png)

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理

### 添加资产

#### 简介
dapp提供按钮给用户， 直接将指定的Token添加到用户插件的资产展示列表中


#### 技术规范
##### 代码示例
```typescript
const res = await tronWeb.request(
  {
    method: 'wallet_watchAsset',
    params: {
      type: 'TRC20',
      options: {
          address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t'
      }
    } as WatchAssetParams,
  }
);
```

##### 参数
```typescript
interface WatchAssetParams {
  type: 'trc10' | 'trc20' | 'trc721';
  options: {
    address: string;
    symbol?: string;
    decimals?: number;
    image?: string;
  }
}
```
* method: wallet_watchAsset 固定的字符串
* params: WatchAssetParams，具体参数如下
  * type: 目前只支持'trc10', 'trc20', 'trc721'三种
  * options: 
    * address: token的合约地址 或者 token id, 必传
    * symbol: 占位(目前未使用)，可选 
    * decimals: 占位(目前未使用)，可选 
    * image: 占位(目前未使用)，可选 


##### 返回值
此方法没有返回值

#### 交互流程
##### 添加TRC10资产
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  try {
    tronweb.request({
      method: 'wallet_watchAsset',
      params: {
        type: 'trc10',
        options: {
          address: '1002000'
        },
      },
    });
  } catch (e) {}
}
```

代码执行时，TronLink会弹出添加窗口，用户点击确定添加TRC10资产，或者取消添加。

![image](../images/zh_plugin-wallet_add_trc10.png)

点击”添加”按钮，资产被添加到资产列表，如下图所示。

![image](../images/zh_plugin-wallet_add_trc10_success.png)



##### 添加TRC20资产
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  try {
    tronweb.request({
      method: 'wallet_watchAsset',
      params: {
        type: 'trc20',
        options: {
          address: 'TN3W4H6rK2ce4vX9YnFQHwKENnHjoxb3m9'
        },
      },
    });
  } catch (e) {}
}
```

代码执行时，TronLink会弹出添加窗口，用户点击确定添加TRC20资产，或者取消添加。

![image](../images/zh_plugin-wallet_add_trc20.png)

点击”添加”按钮，资产被添加到资产列表，如下图所示。

![image](../images/zh_plugin-wallet_add_trc20_success.png)

##### 添加TRC721资产
```typescript
if (window.tronLink.ready) {
  const tronweb = tronLink.tronWeb;
  try {
    tronweb.request({
      method: 'wallet_watchAsset',
      params: {
        type: 'trc721',
        options: {
          address: 'TVtaUnsgKXhTfqSFRnHCsSXzPiXmm53nZt'
        },
      },
    });
  } catch (e) {}
}
```

代码执行时，TronLink会弹出添加窗口，用户点击确定添加TRC20资产，或者取消添加。

![image](../images/zh_plugin-wallet_add_trc721.png)

点击”添加”按钮，资产被添加到资产列表，如下图所示。

![image](../images/zh_plugin-wallet_add_trc721_success.png)


### 切换网络 TIP-3326

#### 简介
大部分dapp都会在特定的链上提供服务。dapp可以通过调用本协议，告诉TronLink期望使用的链，TronLink会弹出弹窗告知用户即将切换的链，用户可以选择是否同意切换。
在用户同意切换后，dapp可以基于目标链，提供正常的dapp服务。
本协议遵循以太坊EIP-3326协议。

#### 技术规范
##### 代码示例
```javascript
try {
  await tronLink.request({
    method: 'wallet_switchEthereumChain',
    params: [{chainId: '0x2b6653dc'}]
  });
} catch (e) {}
```
##### 参数
params接受一个只有单一元素的数组，单一元素为`SwitchTronChainParameter`类型：
```typescript
interface SwitchTronChainParameter {
  chainId: string;
}
```
- 目前只支持如下chainId：
  - mainnet(主网): `0x2b6653dc`
  - shasta(shasta测试网): `0x94a9059e`
  - nile(nile测试网): `0xcd8690dc`
- chainId的值大小写敏感。


##### 返回值
如果成功，返回null。
如果失败，则会返回错误码及报错信息。详见“错误码”部分。

##### 错误码
|  错误码   | 名称  | 描述 |
|  ----  | ----  | ---- |
| 4001  | 用户拒绝请求 | 用户通过点击“拒绝”按钮，或关闭弹窗，都会触发该错误码 |
| 4902  | 不合法chainId | 目前只支持部分chainId，详见“目前支持的chainId”部分 |
| -32002  | 处于其他流程中 | 当前dapp存在其他流程，无法执行当前请求 |
| -32602  | 参数不合规 | 传入的参数不合规，或者传入了额外的参数 |
| 4200  | 不支持此方法 | 不支持此方法 |


#### 交互流程
触发`wallet_switchEthereumChain`之后，如果TronLink处于锁屏状态，会弹出锁屏弹窗：

![image](../images/zh_plugin-wallet_lock-page.png)

解锁后，或者TronLink已经提前解锁，会打开连接确认的弹窗：

![image](../images/zh_plugin-wallet_switch-chain.png)

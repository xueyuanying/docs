# 被动接收TronLink插件的消息

TronLink 支持 TRON 主网及测试网（Shasta、Nile）。开发者可以在 DApp 中监听 TronLink 派发的事件消息，以判断当前选择的网络以及当前激活的账户。下面通过一个简单示例进行演示。

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TronLink Events Demo</title>
</head>
<body>
<script>
    // 1. 等待 TronLink 注入 window.tronLink 后注册现代 API 监听
    function handleTronLink() {
        if (!window.tronLink) {
            console.log("请安装 TronLink 扩展！");
            return;
        }
        console.log("tronLink 检测成功！");

        // 现代 API（window.tron.on）— 推荐使用
        window.tron.on("accountsChanged", (accounts) => {
            console.log("accountsChanged", accounts); // 钱包锁定时为 []
        });
        window.tron.on("chainChanged", ({ chainId }) => {
            console.log("chainChanged", chainId);
        });
        window.tron.on("connect", ({ chainId }) => {
            console.log("connect", chainId);
        });
        window.tron.on("disconnect", (err) => {
            console.log("disconnect", err); // { code: 4900, message: 'Disconnected' }
        });
    }

    if (window.tronLink) {
        handleTronLink();
    } else {
        window.addEventListener("tronLink#initialized", handleTronLink, { once: true });
        setTimeout(handleTronLink, 3000); // 兜底：防止事件已经错过
    }

    // 2. 原始 window.postMessage 事件 — 涵盖旧版 3.x 事件与授权流程
    window.addEventListener("message", function (e) {
        if (!e.data || !e.data.message) return;
        const { action, data } = e.data.message;

        switch (action) {
            // --- 现代授权流程 ---
            case "connectWeb":
                console.log("connectWeb", data);
                break;
            case "acceptWeb":
                console.log("acceptWeb", data);
                break;
            case "rejectWeb":
                console.log("rejectWeb", data);
                break;
            case "disconnectWeb":
                console.log("disconnectWeb", data);
                break;

            // --- 已废弃的 3.x 事件（主链 / 侧链检测） ---
            case "tabReply":
                if (data?.data?.node?.chain === "_") {
                    console.log("tronLink 当前选择主链");
                } else {
                    console.log("tronLink 当前选择侧链");
                }
                break;
            case "setAccount":
                console.log("setAccount，当前地址：", data.address);
                break;
            case "setNode":
                if (data?.node?.chain === "_") {
                    console.log("tronLink 当前选择主链");
                } else {
                    console.log("tronLink 当前选择侧链");
                }
                break;
        }
    });
</script>
</body>
</html>
```

下面分别对每个监听器进行详细说明，含返回值与截图。

---

### 初始化完成事件

事件标识：`tronLink#initialized`

#### 简介

当 TronLink 完成 `window.tronLink` 注入后，会在 `window` 上派发该事件。可用它在调用任何钱包方法前安全地等待对象就绪，避免轮询。

#### 技术规范

##### 代码示例

```javascript
if (window.tronLink) {
  handleTronLink();
} else {
  window.addEventListener('tronLink#initialized', handleTronLink, {
    once: true,
  });
  // 兜底：防止事件已经错过
  setTimeout(handleTronLink, 3000);
}

function handleTronLink() {
  const { tronLink } = window;
  if (tronLink) {
    console.log('tronLink successfully detected!');
  } else {
    console.log('Please install TronLink-Extension!');
  }
}
```

### 账户改变消息

消息标识： `accountsChanged`
#### 简介
以下情况会产生此消息

  1. 用户登录

  2. 用户切换账号

  3. 用户锁定钱包

  4. 钱包超时自动锁定

#### 技术规范
##### 代码示例
```typescript
window.tron.on('accountsChanged', (accountArray) => {
  // handler logic
  console.log('got accountsChanged event', accountArray)
})
```
##### 返回值
```typescript
['your_current_account_address']
```
###### 返回值示例
1. 用户登录时，消息体内容为
```json
['TZ5XixnRyraxJJy996Q1sip85PHWuj4793'] // 当前的账号
```

2. 用户切换账号时，消息体内容为
```json
['TRKb2nAnCBfwxnLxgoKJro6VbyA6QmsuXq'] // 新选择的账号地址
```

3. 用户锁定和钱包超时自动锁定时，消息体内容为
```json
[]
```


### 网络改变消息

消息标识： `chainChanged`
#### 简介
开发者可以监听此消息来获取网络的改变
以下情况会产生此消息

1. 用户改变网络的时候
#### 技术规范
##### 代码示例
```typescript
window.tron.on('chainChanged', ({chainId}) => {
  // handler logic
  console.log('got chainChanged event', chainId)
})
```

##### 返回值

```json
{
  chainId: string;
}
```
- 目前只支持如下chainId：
  - mainnet(主网): `0x2b6653dc`
  - shasta(shasta测试网): `0x94a9059e`
  - nile(nile测试网): `0xcd8690dc`
- chainId的值大小写敏感。

### TronLink可以提供服务的消息

消息标识： `connect`
#### 简介
如果TronLink及`window.tron`对象可用，那么必定发出此事件。
这包括以下情况：

 - provider在初始化完成后，连接到一个链。

 - `disconnect`事件发出后，provider连接到一个链。

#### 技术规范
##### 代码示例
```typescript
window.tron.on('connect', ({chainId}) => {
  // handler logic
  console.log('got connect event', chainId)
})
```

### 断开连接网站消息
消息标识： `disconnect`
#### 简介
如果provider与所有链断开连接，则provider必须发出`disconnect`事件，并返回`ProviderRpcError`对象的错误。
#### 技术规范
##### 代码示例
```typescript
tron.on('disconnect', (providerRpcError: ProviderRpcError) => {
  console.error(connectInfo); // { code: 4900, message: 'Disconnected' }
})
```

以下消息通过 `window.postMessage` 派发，DApp 接收到的内容是一个 `MessageEvent`，事件结构可参考 [MessageEvent 的 MDN 文档](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent)。

### 历史遗留问题
以下四个消息为了兼容3.x保留，并在未来版本将会被废弃

1. 用户拒绝连接消息`rejectWeb`

2. 用户断连网站消息`disconnectWeb`

3. 用户确定连接消息`acceptWeb`

4. 用户主动连接网站消息`connectWeb`

#### 用户拒绝连接消息
消息标识： `rejectWeb`

以下情况会产生此消息

1. dapp请求连接，用户在弹窗中拒绝连接后
 
![image](../images/zh_plugin-wallet_rejectWeb.png)


开发者可以监听此消息来获取用户拒绝连接消息
```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "rejectWeb") {
      // handler logic
      console.log('got rejectWeb event', e.data)
  }
})
```

#### 用户断连网站消息
消息标识： `disconnectWeb`

以下情况会产生此消息

1. 用户主动断开连接

![image](../images/zh_plugin-wallet_disconnectWeb.png)


开发者可以监听此消息来获取用户主动断连消息
```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "disconnectWeb") {
      // handler logic
      console.log('got disconnectWeb event', e.data)
  }
})
```

#### 用户确定连接消息
消息标识： `acceptWeb`

以下情况会产生此消息

1. 用户确定连接消息


![image](../images/zh_plugin-wallet_acceptWeb.png)

开发者可以监听此消息来获取用户确定连接消息
```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "acceptWeb") {
      // handler logic
      console.log('got acceptWeb event', e.data)
  }
})
```

#### 用户主动连接网站消息
消息标识： `connectWeb`

以下情况会产生此消息

1. 用户主动连接网站时


![image](../images/zh_plugin-wallet_connectWeb.png)


开发者可以监听此消息来获取用户主动连接网站消息
```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "connectWeb") {
      // handler logic
      console.log('got connectWeb event', e.data)
  }
})
```

---

### 已废弃的 3.x 事件（主链 / 侧链）

以下事件是 TronLink 3.x 通过原始 `window.postMessage` 用于检测当前网络与账户的方式。它们已被 `window.tron` 上的 `accountsChanged` / `chainChanged` 取代，仅供存量代码参考 — 新接入请勿使用。

#### 网络初始化（`tabReply`）

页面加载后 TronLink 完成初始化时触发一次。检查 `e.data.message.data.data.node.chain`：值为 `'_'` 表示主链，其他值表示对应侧链标识。

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "tabReply") {
    if (e.data.message.data.data.node.chain == '_') {
      console.log("tronLink 当前选择主链");
    } else {
      console.log("tronLink 当前选择侧链");
    }
  }
});
```

#### 账户切换（`setAccount`）

用户切换当前账户时触发。

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "setAccount") {
    console.log("当前地址：", e.data.message.data.address);
  }
});
```

#### 网络切换（`setNode`）

用户切换网络时触发。`chain == '_'` 的判断方式与 `tabReply` 相同。

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "setNode") {
    if (e.data.message.data.node.chain == '_') {
      console.log("tronLink 当前选择主链");
    } else {
      console.log("tronLink 当前选择侧链");
    }
  }
});
```
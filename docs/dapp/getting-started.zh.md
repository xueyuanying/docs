# 开始开发

本文档以端到端方式带你完成 DApp 与 TronLink 钱包的对接 — 从检测钱包、请求授权，到签名并广播一笔交易。

## 概述

TronLink 是一款面向 TRON 网络的浏览器扩展钱包。安装 TronLink 后，DApp 无需自行运行全节点即可与 TRON 网络交互 — TronLink 会向每个页面注入 `window.tronLink` 对象，对外暴露一个 `tronWeb` 实例以及与钱包通信所需的 `request` 方法。

## `window.tronLink` 对象

```typescript
interface tronLinkParams {
  ready: boolean;             // 用户授权前为 false，授权后为 true
  request: (args: any) => any; // DApp 调用钱包能力的方法
  tronWeb: TronWeb;           // 与当前账户/网络绑定的 TronWeb 实例
}
```

用户完成授权后，`window.tronLink.tronWeb` 即可用于构建、签名与广播交易。

## 检测 TronLink（TIP-6963）

TIP-6963 是检测 TronLink（及其他 TRON 钱包）的推荐方式，避免污染全局命名空间。监听 `TIP6963:announceProvider` 事件，并派发 `TIP6963:requestProvider` 让已安装的钱包主动声明自己。

```javascript
let provider;

window.addEventListener("TIP6963:announceProvider", (e) => {
  if (e.detail.info && e.detail.info.name === "TronLink") {
    provider = e.detail.provider;
  }
});

window.dispatchEvent(new Event("TIP6963:requestProvider"));
```

如果派发请求后 `provider` 仍为 undefined，则说明用户未安装 TronLink，可提示用户进行安装。

## 请求授权

通过 `tron_requestAccounts` 请求用户授权连接钱包，返回值会指明用户是否同意、请求是否在排队、或被拒绝。

```typescript
interface RequestAccountsResponse {
  code: number;     // 200: 同意，4000: 已在队列中，4001: 用户拒绝
  message: string;
}

const res: RequestAccountsResponse = await tronLink.request({
  method: "tron_requestAccounts",
});
```

| code | 说明 |
| ---- | ---- |
| 200  | 用户同意授权 |
| 4000 | 已在队列中，请勿重复提交 |
| 4001 | 用户拒绝授权 |

完整 TIP-1102 规范、错误码及旧版连接方式请参考 [主动请求 TronLink 插件功能](../plugin-wallet/active-requests.md)。

## 获取 `tronWeb` 实例

最简化的连接逻辑 — 如果用户已授权过当前 DApp，直接复用现有连接，否则发起授权请求。

```javascript
async function getTronWeb() {
  let tronWeb;
  if (window.tronLink.ready) {
    tronWeb = tronLink.tronWeb;
  } else {
    const res = await tronLink.request({ method: "tron_requestAccounts" });
    if (res.code === 200) {
      tronWeb = tronLink.tronWeb;
    }
  }
  return tronWeb;
}
```

获取 `tronWeb` 实例后，即可执行链上交互：TRX/TRC20 转账、多签交易、消息签名、合约调用等。完整的 `tronWeb` API 用法请参考 [TronWeb 文档](https://tronweb.network/docu/docs/intro)。

## 端到端示例：签名并广播一笔 TRX 转账

下面这个独立的 HTML 示例通过 TIP-6963 检测 TronLink、请求授权、构建一笔 TRX 转账、在 TronLink 弹窗中提示用户签名，然后将签名后的交易广播到链上。

```html
<!DOCTYPE html>
<html lang="zh">
  <head>
    <meta charset="UTF-8" />
    <title>TronLink Demo</title>
  </head>
  <body>
    <button onclick="connect()">连接 TronLink</button>
    <script>
      let provider;

      window.addEventListener("TIP6963:announceProvider", (e) => {
        if (e.detail.info && e.detail.info.name === "TronLink") {
          provider = e.detail.provider;
        }
      });
      window.dispatchEvent(new Event("TIP6963:requestProvider"));

      async function connect() {
        if (!provider) return alert("未检测到 TronLink！");
        try {
          const res = await provider.request({
            method: "tron_requestAccounts",
          });
          if (res.code !== 200) {
            console.warn("授权失败：", res);
            return;
          }

          const tronweb = provider.tronWeb;
          const from = tronweb.defaultAddress.base58;
          const to = "TN9RRaXkCFtTXRso2GdTZxSxxwufzxLQPP";

          // 构建一笔 10 SUN 的 TRX 转账（金额单位为 SUN：1 TRX = 1_000_000 SUN）
          const tx = await tronweb.transactionBuilder.sendTrx(to, 10, from);

          // 弹出 TronLink 确认窗口
          const signedTx = await tronweb.trx.sign(tx);

          // 广播到网络
          const broadcastTx = await tronweb.trx.sendRawTransaction(signedTx);
          console.log(broadcastTx);
        } catch (error) {
          console.error(error);
        }
      }
    </script>
  </body>
</html>
```

## 相关文档

- [主动请求 TronLink 插件功能](../plugin-wallet/active-requests.md) — 完整的请求/响应说明
- [被动接收 TronLink 插件的消息](../plugin-wallet/passive-messages.md) — 账户 / 网络变化事件
- [普通转账](transfer.md)、[多签转账](multi-sign-transfer.md)、[消息签名](message-signing.md)、[Stake 2.0](stake2.md)
- [TronWeb 文档](https://tronweb.network/docu/docs/intro)
- 参考：[https://developers.tron.network/docs/introduction](https://developers.tron.network/docs/introduction)

# 普通转账

**简介**

DApp 需要用户发起一笔 TRX 转账。

> **前提条件：** 已通过 `eth_requestAccounts` 完成 DApp 连接授权（参见[开始开发](getting-started.md)）。

波场网络上发起转账需要3个步骤：

  1. 构造转账交易

  2. 对交易进行签名

  3. 对签名后的交易进行广播

在这里，TronLink 介入的是第2步签名的部分，1, 3 两步需要开发者使用 tronWeb 完成

**技术规范**

**代码示例**

```javascript
const tronweb = window.tron.tronWeb;
const fromAddress = tronweb.defaultAddress.base58;
const toAddress = "TTSFjEG3Lu9WkHdp4JrWYhbGP6K1REqnGQ";
const tx = await tronweb.transactionBuilder.sendTrx(toAddress, 10, fromAddress); // 步骤1
try {
  const signedTx = await tronweb.trx.sign(tx); // 步骤2
  await tronweb.trx.sendRawTransaction(signedTx); // 步骤3
} catch (e) {
  // error handling
}
```

当代码执行到`await tronweb.trx.sign(tx);`时，TronLink钱包会提示弹窗，需要用户进行确认，如下图：

![image](../images/zh_dapp_pu-tong-zhuan-zhang_img_0.jpg)

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理。

如果用户在弹窗中选择【签名】，DApp 可以拿到签名后的交易，继续进行广播。


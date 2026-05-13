# 多签转账

**简介**

此处可参考[普通转账](transfer.zh.md)

> **前提条件：** 已通过 `eth_requestAccounts` 完成 DApp 连接授权（参见[开始开发](getting-started.md)）。

**技术规范**

**代码示例**

```javascript
const tronweb = window.tron.tronWeb;
const toAddress = "TRKb2nAnCBfwxnLxgoKJro6VbyA6QmsuXq";
const activePermissionId = 2;
const tx = await tronweb.transactionBuilder.sendTrx(
    toAddress, 10,
    { permissionId: activePermissionId }
); // 步骤1
try {
  const signedTx = await tronweb.trx.multiSign(tx, undefined, activePermissionId); // 步骤2
  await tronweb.trx.sendRawTransaction(signedTx); // 步骤3
} catch (e) {}
```

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理。

如果用户在弹窗中选择【签名】，DApp 可以拿到签名后的交易，继续进行广播。


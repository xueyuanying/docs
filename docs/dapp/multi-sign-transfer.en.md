# Multi-Signature Transfer

**Overview**

For this section, you may refer to [General Transfer](transfer.en.md)

> **Prerequisite:** The DApp connection has been authorized via `eth_requestAccounts` (see [Start Developing](getting-started.md#request-authorization)).

**Specification**

**Example**

```javascript
const tronweb = window.tron.tronWeb;
const toAddress = "TRKb2nAnCBfwxnLxgoKJro6VbyA6QmsuXq";
const activePermissionId = 2;
const tx = await tronweb.transactionBuilder.sendTrx(
  toAddress, 10,
  { permissionId: activePermissionId }
); // Step 1
try {
  const signedTx = await tronweb.trx.multiSign(tx, undefined, activePermissionId); // Step 2
  await tronweb.trx.sendRawTransaction(signedTx); // Step 3
} catch (e) {}
```

If the user chooses “Reject” in the pop-up window, an exception will be thrown, which the developer can catch for further processing. If the user chooses “Sign” in the pop-up window, the DApp receives and broadcasts the signed transaction.



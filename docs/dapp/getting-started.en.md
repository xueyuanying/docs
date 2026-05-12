# Start Developing

This document walks you through connecting a DApp to the TronLink wallet end-to-end — from detecting the wallet, requesting authorization, to signing and broadcasting a transaction.

## Overview

TronLink is a browser extension wallet for the TRON network. By integrating TronLink, your DApp can communicate with the TRON network — TronLink injects a `window.tron` object into every page, exposing a `tronWeb` instance and a `request` method for talking to the wallet.

## The `window.tron` Object

```typescript
interface TronProvider {
  isTronLink: true;
  request: (args: { method: string; params?: any }) => Promise<any>; // Method used by the DApp to invoke wallet features
  tronWeb: TronWeb;          // TronWeb instance bound to the user's current account/network
}
```

Once the user authorizes the connection, `window.tron.tronWeb` is fully usable for building, signing and broadcasting transactions.

## Detect TronLink (TIP-6963)

TIP-6963 is the recommended way to detect TronLink (and other TRON wallets) without polluting the global namespace. Listen for `TIP6963:announceProvider` events and dispatch a `TIP6963:requestProvider` to ask installed wallets to announce themselves.

```javascript
let provider;

window.addEventListener("TIP6963:announceProvider", (e) => {
  if (e.detail.info && e.detail.info.name === "TronLink") {
    provider = e.detail.provider;
  }
});

window.dispatchEvent(new Event("TIP6963:requestProvider"));
```

If `provider` is still undefined after dispatching the request, TronLink is not installed — prompt the user to install it.

## Request Authorization

Use `eth_requestAccounts` to ask the user to connect their wallet. On approval the promise resolves with an array containing the selected address; on failure it rejects with a `{ code, message }` error.

```typescript
try {
  const accounts: string[] = await tron.request({ method: "eth_requestAccounts" });
  console.log("Connected:", accounts[0]);
} catch (err) {
  // err shape: { code: number, message: string }
  console.warn("Authorization failed:", err);
}
```

| code   | Description |
| ------ | ----------- |
| 4001   | User rejected the request (clicked Reject, closed the popup, or the request did not come from the active tab) |
| -32000 | DApp requests are too frequent (rate-limited while the wallet is locked) |
| 4200   | Unsupported method |

For the full TIP-1102 specification, error codes and the legacy connection method, see [Proactively Request TronLink Plugin Features](../plugin-wallet/active-requests.md).

## Get the `tronWeb` Instance

The simplest connection helper — reuses the existing connection if the user already authorized this DApp, otherwise requests authorization first.

```javascript
async function getTronWeb() {
  if (window.tron.tronWeb?.ready) {
    return window.tron.tronWeb;
  }
  await window.tron.request({ method: "eth_requestAccounts" });
  return window.tron.tronWeb;
}
```

After obtaining the `tronWeb` instance you can perform on-chain interactions: TRX/TRC20 transfers, multi-signature transactions, message signing, contract calls, and so on. For full `tronWeb` API usage, see the [TronWeb documentation](https://tronweb.network/docu/docs/intro).

## End-to-End Example: Sign and Broadcast a TRX Transfer

The following self-contained HTML example detects TronLink via TIP-6963, requests authorization, builds a TRX transfer, prompts the user to sign in the TronLink popup, and broadcasts the signed transaction.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>TronLink Demo</title>
  </head>
  <body>
    <button onclick="connect()">Connect TronLink</button>
    <script>
      let provider;

      window.addEventListener("TIP6963:announceProvider", (e) => {
        if (e.detail.info && e.detail.info.name === "TronLink") {
          provider = e.detail.provider;
        }
      });
      window.dispatchEvent(new Event("TIP6963:requestProvider"));

      async function connect() {
        if (!provider) return alert("TronLink not found!");
        try {
          const accounts = await provider.request({
            method: "eth_requestAccounts",
          });
          console.log("Connected:", accounts[0]);

          const tronweb = provider.tronWeb;
          const from = tronweb.defaultAddress.base58;
          const to = "TN9RRaXkCFtTXRso2GdTZxSxxwufzxLQPP";

          // Build a 10 SUN TRX transfer (amount unit is SUN: 1 TRX = 1_000_000 SUN)
          const tx = await tronweb.transactionBuilder.sendTrx(to, 10, from);

          // Pops up the TronLink confirmation window
          const signedTx = await tronweb.trx.sign(tx);

          // Broadcast to the network
          const broadcastTx = await tronweb.trx.sendRawTransaction(signedTx);
          console.log(broadcastTx);
        } catch (error) {
          // { code: 4001, message: "User rejected the request." } when the user declines
          console.error(error);
        }
      }
    </script>
  </body>
</html>
```

## Related Documentation

- [Proactively Request TronLink Plugin Features](../plugin-wallet/active-requests.md) — full request/response reference
- [Receive Messages from TronLink](../plugin-wallet/passive-messages.md) — account / network change events
- [General Transfer](transfer.md), [Multi-Signature Transfer](multi-sign-transfer.md), [Message Signature](message-signing.md), [Stake 2.0](stake2.md)
- [TronWeb Documentation](https://tronweb.network/docu/docs/intro)

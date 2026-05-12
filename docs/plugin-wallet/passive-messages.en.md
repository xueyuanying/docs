# Passively Receiving Messages from the TronLink Plugin

TronLink supports the TRON mainnet and testnets (Shasta, Nile). Developers can listen for the event messages dispatched by TronLink in their DApp to detect the currently selected network and the currently active account. Let's learn it with a simple example.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TronLink Events Demo</title>
</head>
<body>
<script>
    // 1. Wait for TronLink to inject window.tronLink, then wire up modern listeners
    function handleTronLink() {
        if (!window.tronLink) {
            console.log("Please install TronLink-Extension!");
            return;
        }
        console.log("tronLink successfully detected!");

        // Modern API (window.tron.on) — preferred
        window.tron.on("accountsChanged", (accounts) => {
            console.log("accountsChanged", accounts); // [] when locked
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
        setTimeout(handleTronLink, 3000); // fallback in case the event was missed
    }

    // 2. Raw window.postMessage events — covers legacy 3.x events and authorization flow
    window.addEventListener("message", function (e) {
        if (!e.data || !e.data.message) return;
        const { action, data } = e.data.message;

        switch (action) {
            // --- Legacy authorization events (3.x compat, deprecated in future versions) ---
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

            // --- Deprecated 3.x events (mainchain / sidechain detection) ---
            case "tabReply":
                if (data?.data?.node?.chain === "_") {
                    console.log("tronLink currently selects the main chain");
                } else {
                    console.log("tronLink currently selects the side chain");
                }
                break;
            case "setAccount":
                console.log("setAccount, current address:", data.address);
                break;
            case "setNode":
                if (data?.node?.chain === "_") {
                    console.log("tronLink currently selects the main chain");
                } else {
                    console.log("tronLink currently selects the side chain");
                }
                break;
        }
    });
</script>
</body>
</html>
```

The remainder of this page documents each listener individually, including return values and screenshots.

---

### Initialization Event

Event identifier: `tronLink#initialized`

#### Overview

Fired on `window` after TronLink finishes injecting `window.tronLink` into the page. Use it to safely wait for the wallet object before calling any of its methods, instead of polling.

#### Technical Specification

##### Code Example

```javascript
if (window.tronLink) {
  handleTronLink();
} else {
  window.addEventListener('tronLink#initialized', handleTronLink, {
    once: true,
  });
  // Fallback in case the event was missed
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


### Account Change Message

Message identifier: `accountsChanged`

#### Overview

This message is generated in the following situations:

1. User logs in  
2. User switches account  
3. User locks the wallet  
4. Wallet auto-locks due to timeout  

#### Technical Specification

##### Code Example

```typescript
window.tron.on('accountsChanged', (accountArray) => {
  // handler logic
  console.log('got accountsChanged event', accountArray)
})
```

##### Return Value

```typescript
['your_current_account_address']
```

###### Return Value Examples

1. When the user logs in:

```json
['TZ5XixnRyraxJJy996Q1sip85PHWuj4793']
```

(Current account)

2. When the user switches accounts:

```json
['TRKb2nAnCBfwxnLxgoKJro6VbyA6QmsuXq']
```

(Newly selected account)

3. When the wallet is locked or auto-locked:

```json
[]
```


### Network Change Message

Message identifier: `chainChanged`

#### Overview

Developers can listen to this message to detect network changes.

This message is generated when:

1. The user changes the network.

#### Technical Specification

##### Code Example

```typescript
window.tron.on('chainChanged', ({chainId}) => {
  // handler logic
  console.log('got chainChanged event', chainId)
})
```

##### Return Value

```json
{
  chainId: string;
}
```

Currently supported `chainId` values:

- Mainnet: `0x2b6653dc`
- Shasta Testnet: `0x94a9059e`
- Nile Testnet: `0xcd8690dc`

**Note:** Chain IDs are case-sensitive.

### TronLink Service Availability Message

Message identifier: `connect`

#### Overview

If TronLink and the `window.tron` object are available, this event will always be emitted.

This includes:

- After the provider initializes and connects to a chain.  
- After a `disconnect` event when the provider reconnects to a chain.  

#### Technical Specification

##### Code Example

```typescript
window.tron.on('connect', ({chainId}) => {
  // handler logic
  console.log('got accountsChanged event', chainId)
})
```


### Website Disconnect Message

Message identifier: `disconnect`

#### Overview

If the provider disconnects from all chains, it must emit a `disconnect` event and return a `ProviderRpcError` object.

#### Technical Specification

##### Code Example

```typescript
tron.on('disconnect', (providerRpcError: ProviderRpcError) => {
  console.error(connectInfo); // { code: 4900, message: 'Disconnected' }
})
```

---

The remaining messages are dispatched via `window.postMessage`. The content received by a DApp is a `MessageEvent` — see the [MessageEvent MDN documentation](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent) for the event shape.

### Legacy Compatibility Messages

The following four messages are retained for compatibility with version 3.x and will be deprecated in future versions:

1. User rejected connection → `rejectWeb`  
2. User disconnected the website → `disconnectWeb`  
3. User confirmed connection → `acceptWeb`  
4. User proactively connected the website → `connectWeb`  

### User Rejected Connection Message

Message identifier: `rejectWeb`

Generated when:

1. A DApp requests connection and the user rejects it in the popup.

![image](../images/plugin-wallet_rejectWeb.jpg){ width="350" }

Developers can listen for this message:

```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "rejectWeb") {
      // handler logic
      console.log('got rejectWeb event', e.data)
  }
})
```


### User Disconnected Website Message

Message identifier: `disconnectWeb`

Generated when:

1. The user manually disconnects the website.

![image](../images/plugin-wallet_disconnectWeb.jpg){ width="350" }

Developers can listen for this message:

```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "disconnectWeb") {
      // handler logic
      console.log('got disconnectWeb event', e.data)
  }
})
```

### User Confirmed Connection Message

Message identifier: `acceptWeb`

Generated when:

1. The user confirms the connection request.

![image](../images/plugin-wallet_acceptWeb.jpg){ width="350" }

Developers can listen for this message:

```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "acceptWeb") {
      // handler logic
      console.log('got acceptWeb event', e.data)
  }
})
```

### User Proactively Connected Website Message

Message identifier: `connectWeb`

Generated when:

1. The user proactively connects to the website.

![image](../images/plugin-wallet_connectWeb.jpg){ width="350" }

Developers can listen for this message:

```typescript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "connectWeb") {
      // handler logic
      console.log('got connectWeb event', e.data)
  }
})
```

---

### Deprecated 3.x Events (Mainchain / Sidechain)

The following events were used by TronLink 3.x to detect the active network and account via raw `window.postMessage`. They have been superseded by the modern `accountsChanged` / `chainChanged` listeners on `window.tron` and are kept here for reference only — new integrations should not use them.

#### Network Initialization (`tabReply`)

Fired once after page load when TronLink finishes initializing. Inspect `e.data.message.data.data.node.chain` to determine whether the wallet is on the mainchain (`'_'`) or a sidechain (any other identifier).

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "tabReply") {
    if (e.data.message.data.data.node.chain == '_') {
      console.log("tronLink currently selects the main chain");
    } else {
      console.log("tronLink currently selects the side chain");
    }
  }
});
```

#### Account Switch (`setAccount`)

Fired when the user switches the active account.

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "setAccount") {
    console.log("current address:", e.data.message.data.address);
  }
});
```

#### Network Switch (`setNode`)

Fired when the user changes the network. Same `chain == '_'` convention as `tabReply`.

```javascript
window.addEventListener('message', function (e) {
  if (e.data.message && e.data.message.action == "setNode") {
    if (e.data.message.data.node.chain == '_') {
      console.log("tronLink currently selects the main chain");
    } else {
      console.log("tronLink currently selects the side chain");
    }
  }
});
```

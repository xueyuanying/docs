
# 即将实施：对ledger签名后signature的v字段进行兼容改造
针对ledger签名（包括所有交易签名、消息签名、712签名），在将来的几个月内，会更新一项新的技术措施，会将签名后的 `signature` 字段中末尾的`01`替换成`1c`，`00`替换成`1b`。

这个改动，是为了保证ledger账户签名和普通账户签名的末尾两位一致。

### 普通交易签名

以下是签名成功的 ledger 交易体：
```json
{
    "txID": "......",
    .....other property,
    "signature": [
        "......01"
    ]
}
```
会变成:
```json
{
    "txID": "......",
    .....other property,
    "signature": [
        "......1c"
    ]
}
```
如果需要广播，DApp会使用修改后的交易体进行广播

### 消息签名

DApp发送消息签名
```javascript
tron.tronWeb.trx.sign('.....unsign_string');
```
或者
```javascript
tron.tronWeb.trx.signMessageV2('.....unsign_string');
```

ledger返回的实际签名hash：
```
......50f400
```

经过修改后的hash：
```
......50f41b
```

### 712签名

DApp发送712签名
```javascript
const types = {
  Person: [
    {
      name: 'name',
      type: 'string',
    },
    {
      name: 'wallet',
      type: 'address',
    },
  ],
  Mail: [
    {
      name: 'from',
      type: 'Person',
    },
    {
      name: 'to',
      type: 'Person',
    },
    {
      name: 'contents',
      type: 'string',
    },
  ],
};

const primaryType = 'Mail';

const domain = {
  name: 'Ether Mail',
  version: '1',
  chainId: '0x2b6653dc',
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC',
};

const message = {
  from: {
    name: 'Cow',
    wallet: '0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826',
  },
  to: {
    name: 'Bob',
    wallet: '0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB',
  },
  contents: 'Hello, Bob!',
};

await tron.tronWeb.trx._signTypedData(domain, types, message);
```

ledger返回的实际签名hash：
```
.....7ee700
```

经过修改后的hash：
```
......7ee71b
```
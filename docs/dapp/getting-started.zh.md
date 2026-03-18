# 开始开发

本文档将以最简单易懂的方式指导你的 DApp 与 TronLink 应用程序相连接。

在DApp加载完成后，TronLink 会在其中注入 `window.tronLink` 对象，

  1. 如果用户连接过此 DApp， 则可以直接获取`tronLink.tronWeb 。`

  2. 如果未连接过，则可以调用请求连接后获取。

```shell
  async function getTronWeb() {
    let tronWeb;
    if (window.tronLink.ready) {
      tronWeb = tronLink.tronWeb;
    } else {
      const res = await tronLink.request({ method: 'tron_requestAccounts' });
      if (res.code === 200) {
        tronWeb = tronLink.tronWeb;
      }
    }
    return tronWeb;
  }
```

获取 tronWeb 实例后，即可进行 转账签名，多签签名，消息签名等链上交互动作。

具体 tronWeb 实例的使用，可参考以下文档：<a class="tooltip" href="https://tronweb.network/docu/docs/intro" data-tooltip="https://tronweb.network/docu/docs/intro">https://tronweb.network/docu/docs/intro</a>

参考：<a class="tooltip" href="https://developers.tron.network/docs/introduction" data-tooltip="https://developers.tron.network/docs/introduction">https://developers.tron.network/docs/introduction</a>
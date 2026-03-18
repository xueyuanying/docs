# 消息签名

**简介**

DApp 需要用户对一个 hex 消息签名，签名后消息转发给后端进行验签，以此判断用户合法登陆。

**前提**

DApp 开发者完成【连接网站】请求，用户同意连接。

**技术规范**

**代码示例**

```shell

    if (window.tronLink.ready) {
      const tronweb = tronLink.tronWeb;
      try {
        const message = "0x01EF"; // any hex string
        const signedString = await tronweb.trx.signMessageV2(message);
      } catch (e) {}
    }
```

**参数**

`tronLink.tronWeb.trx.signMessageV2`接收一个十六进制的字符串作为参数，该字符串表示当前待签名的内容。

**返回值**

如果用户在弹窗中选择签名, DApp 可以得到签名后的十六进制字符串, 比如：

```shell 
    0xb0e0b150b9b10dc348f25c7f38fc87f16e18c0d230d23946aac519a5ad9e45937f656012d33c09e9d9dec00b03fbb304e797f8991bb823dce676ac91e03a55991b
```

如果报错，则会返回如下信息：

```shell 
    Uncaught (in promise) Invalid transaction provided
```
**交互流程**

当代码执行到`await tronweb.trx.signMessageV2(message);`时，TronLink 钱包会提示弹窗，需要用户进行确认， 如下图, 其中消息内容会以hex的方式展示：

![image](../images/zh_dapp_xiao-xi-qian-ming_img_0.jpg)

如果用户在弹窗中选择【拒绝】，则会抛出异常，开发者可捕获此异常进行业务处理。


# DApp支持

提供流畅的 DApp 浏览器，支持运行 TRON DApp。


## 集成TronLink

TronLink App 内置 DApp 浏览器，对于在其上运行的 TRON DApp，TronLink 会自动将 tronWeb 及 tronLink 对象注入到该 DApp。从而允许 DApp 与 TronLink App，以及TRON 网络交互。

详情：参考[DApp章节](../../dapp/getting-started)

## Dapp 浏览器

### 基本功能

DApp 浏览器支持运行 TRON DApp，并自动注入 tronWeb 及 tronLink 对象

### 扩展

在 DApp 浏览器中运行的 Tron DApp，会自动注入 iTron 对象，并提供 App 端定制化功能

  * 切换屏幕方向

```shell   
        // url: DApp page url
        // screenModel: "1" -> 竖屏；"2" -> 横屏
        public void setScreenModel(String url, String screenModel)
```


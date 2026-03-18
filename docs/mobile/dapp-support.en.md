# DApp Support

## TronLink Integration

TronLink injects a version of TronWeb into the DApp that runs in TronLink's DApp Explorer. This enables the DApp to interact with TronLink DApps and the TRON network.

Details: [Go to DApp](../../dapp/getting-started)


## DApp Explorer

### Basic Function

The DApp Explorer allows Tron DApps to run and automatically injects tronWeb and TronLink objects.

### Extension

Tron DApp running in the DApp Explorer injects iTron objects automatically to offer customized App service.

##### Change screen orientation


```shell     
  // url: DApp page url
  // screenModel: "1" -> vertical;"2" -> horizontal
  void setScreenModel(String url, String screenModel)
```


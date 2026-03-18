# Stake2.0

When generating stake 2.0 transactions for DApps, for transactions of the `DelegateResourceContract` or `UnDelegateResourceContract` type, in order to display the estimated results during signature using the Tronlink extension, it is necessary to add the "__options" field to the transaction body.

Inside "__options", there are two values: estimatedBandwidth and estimatedEnergy, which correspond to the estimated energy and bandwidth of the delegate and reclaim operations, respectively.

When generating stake 2.0 transactions using a non-tronlink extension injected tronweb, in order to display the corresponding type of resource for `DelegateResourceContract` or `UnDelegateResourceContract` type transactions during signature, the "__resource" field needs to be added to the transaction body. (Adding “resource” is only necessary for tronWeb that is not injected by a tronlink extension. TronWeb that is injected by tronlink extension does not require.)

The "__resource" field corresponds to the values "BANDWIDTH" and "ENERGY".

![image](../images/dapp_stake2.0_img_0.jpg)

<style>
img {
  max-width: 100%!important;
}
</style>

For example:

```shell 
    const transaction = await tronWeb.transactionBuilder.delegateResource(10e6, 'receiverAddress', 'BANDWIDTH', 'ownerAddress', false);
    transaction.raw_data.contract[0].parameter.value.resource = "BANDWIDTH"
    transaction.__options = {"estimatedBandwidth": 1}
```

The specific calculation logic of estimatedEnergy and estimatedBandwidth can be found in the last chapter of the "[<a class="tooltip" href="https://coredevs.medium.com/stake-2-0-adaption-faq-66bafdf53606" data-tooltip="https://coredevs.medium.com/stake-2-0-adaption-faq-66bafdf53606">Stake 2.0 Adaptation FAQ</a>](https://coredevs.medium.com/stake-2-0-adaption-faq-66bafdf53606)": "How to convert resource share to amount?"


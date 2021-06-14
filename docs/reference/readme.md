The JavaScript DigitalBits SDK facilitates integration with the [DigitalBits Frontier API server](https://github.com/xdbfoundation/go/tree/master/services/frontier) and submission of DigitalBits transactions, either on Node.js or in the browser. It has two main uses: [querying Frontier](#querying-frontier) and [building, signing, and submitting transactions to the DigitalBits network](#building-transactions).

- [Building and installing js-digitalbits-sdk](https://github.com/xdbfoundation/js-digitalbits-sdk)
- [Examples of using js-digitalbits-sdk](https://developers.digitalbits.io/reference/js-digitalbits-sdk/docs/reference/examples)

# Querying Frontier
js-digitalbits-sdk gives you access to all the endpoints exposed by Frontier.

## Building requests
js-digitalbits-sdk uses the [Builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) to create the requests to send
to Frontier. Starting with a [server](https://digitalbitsorg.github.io/js-digitalbits-sdk/Server.html) object, you can chain methods together to generate a query.
(See the [Frontier reference](https://developers.digitalbits.io/reference/) documentation for what methods are possible.)

```js
var DigitalBitsSdk = require('digitalbits-sdk');
var server = new DigitalBitsSdk.Server('https://frontier.testnet.digitalbits.io');
// get a list of transactions that occurred in ledger 1400
server.transactions()
    .forLedger(1400)
    .call().then(function(r){ console.log(r); });

// get a list of transactions submitted by a particular account
server.transactions()
    .forAccount('GASOCNHNNLYFNMDJYQ3XFMI7BYHIOCFW3GJEOWRPEGK2TDPGTG2E5EDW')
    .call().then(function(r){ console.log(r); });
```

Once the request is built, it can be invoked with `.call()` or with `.stream()`. `call()` will return a
[promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) to the response given by Frontier.

## Streaming requests
Many requests can be invoked with `stream()`. Instead of returning a promise like `call()` does, `.stream()` will return an `EventSource`.
Frontier will start sending responses from either the beginning of time or from the point specified with `.cursor()`.
(See the [Frontier reference](https://developers.digitalbits.io/reference/) documentation to learn which endpoints support streaming.)

For example, to log instances of transactions from a particular account:

```javascript
var DigitalBitsSdk = require('digitalbits-sdk')
var server = new DigitalBitsSdk.Server('https://frontier.testnet.digitalbits.io');
var lastCursor=0; // or load where you left off

var txHandler = function (txResponse) {
    console.log(txResponse);
};

var es = server.transactions()
    .forAccount(accountAddress)
    .cursor(lastCursor)
    .stream({
        onmessage: txHandler
    })
```

## Handling responses

### XDR
The transaction endpoints will return some fields in raw [XDR](https://developers.digitalbits.io/reference/go/services/frontier/internal/docs/reference/xdr)
form. You can convert this XDR to JSON using the `.fromXDR()` method.

An example of re-writing the txHandler from above to print the XDR fields as JSON:

```javascript
var txHandler = function (txResponse) {
    console.log( JSON.stringify(DigitalBitsSdk.xdr.TransactionEnvelope.fromXDR(txResponse.envelope_xdr, 'base64')) );
    console.log( JSON.stringify(DigitalBitsSdk.xdr.TransactionResult.fromXDR(txResponse.result_xdr, 'base64')) );
    console.log( JSON.stringify(DigitalBitsSdk.xdr.TransactionMeta.fromXDR(txResponse.result_meta_xdr, 'base64')) );
};

```


### Following links
The links returned with the Frontier response are converted into functions you can call on the returned object.
This allows you to simply use `.next()` to page through results. It also makes fetching additional info, as in the following example, easy:

```js
server.payments()
    .limit(1)
    .call()
    .then(function(response){
        // will follow the transactions link returned by Frontier
        response.records[0].transaction().then(function(txs){
            console.log(txs);
        });
    });
```


# Transactions

## Building transactions

See the [Building Transactions](https://developers.digitalbits.io/reference/js-digitalbits-base/docs/reference/building-transactions) guide for information about assembling a transaction.

## Submitting transactions
Once you have built your transaction, you can submit it to the DigitalBits network with `Server.submitTransaction()`.

```js
var DigitalBitsSdk = require('digitalbits-sdk')
DigitalBitsSdk.Network.useTestNetwork();
var server = new DigitalBitsSdk.Server('https://frontier.testnet.digitalbits.io');

var transaction = new DigitalBitsSdk.TransactionBuilder(account)
        // this operation funds the new account with XLM
        .addOperation(DigitalBitsSdk.Operation.payment({
            destination: "GASOCNHNNLYFNMDJYQ3XFMI7BYHIOCFW3GJEOWRPEGK2TDPGTG2E5EDW",
            asset: DigitalBitsSdk.Asset.native(),
            amount: "20000000"
        }))
        .build();

transaction.sign(DigitalBitsSdk.Keypair.fromSecret(secretString)); // sign the transaction

server.submitTransaction(transaction)
    .then(function (transactionResult) {
        console.log(transactionResult);
    })
    .catch(function (err) {
        console.error(err);
    });
```

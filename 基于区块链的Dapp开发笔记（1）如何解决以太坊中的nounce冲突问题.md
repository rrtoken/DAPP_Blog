# 基于区块链的Dapp开发笔记（1）
## 如何解决以太坊中的nounce冲突问题

近期我们做了一个小的基于以太坊的DAPP-告白世界，一共4个独立页面（见下图），其中主要是将告白信息上链，来做到对告白信息永久保存。

[![P98TEt.th.jpg](https://s1.ax1x.com/2018/06/24/P98TEt.th.jpg)](https://imgchr.com/i/P98TEt) [![P98H4f.th.jpg](https://s1.ax1x.com/2018/06/24/P98H4f.th.jpg)](https://imgchr.com/i/P98H4f) [![P987UP.th.jpg](https://s1.ax1x.com/2018/06/24/P987UP.th.jpg)](https://imgchr.com/i/P987UP) [![P98IHI.th.jpg](https://s1.ax1x.com/2018/06/24/P98IHI.th.jpg)](https://imgchr.com/i/P98IHI)


这个DAPP我们使用了web3js。我们并没有设置自己以太坊全节点，所以使用使用了infura提供的HttpProvider。
```
const web3 = new Web3(https://mainnet.infura.io/<infuraAccessToken>)
```
由于infura并没有提供private key的存储，所以默认的HttpProvider并不能发起交易，只能进行查询操作。要进行交易有两个选择，第一是使用一个能保存private key并签名交易的provider，比如 truffle-hdwallet-provider。
但是这个provider要求我们将助记词传递给它。
```
var HDWalletProvider = require("truffle-hdwallet-provider");
var provider = new HDWalletProvider(mnemonic, "https://mainnet.infura.io/<infuraAccessToken>");

```
这样做一方面感觉不太放心，另外一方面也不够灵活。所以我们采用了第二个办法，自己签名交易并发送。

自己签名交易的时候必须填写每个交易的nonce值。
由于以太坊设计的技术特点，要求从同一个账号产生的每个交易都有一个不同的nonce，对交易进行区分。这个nonce并不是随意选择的，而是必须从0开始递增。而且每个被以太坊网络记录的交易的nonce都必须比该账号产生的前一个交易大1。

听起来很简单，同时，web3也提供了一个接口，getTransactionCount 让我们查询一个特定账号在网络中已经确认了多少笔交易。所以一个最简单的产生nonce的策略就是使用 `getTransactionCount()` 的返回值。
```
async function sendTransaction(data, account) {
    data.nonce = web.eth.getTransactionCount(account);
    await signAndSend(data);
}
```

不幸的是，以太网是一个弱一致性分布式系统。这里面有太多的不确定性。
试想，由于以太坊网络确认一笔交易需要数分钟的时间。如果在一个很短的时间内（比如10秒之内）我们产生了两笔交易，我们连续调用了两次 `getTransactionCount()` 来产生两个nonce。**我们会惊奇的发现两个nonce会是完全一样**，因为系统根本还没有来得及确认上一笔交易。那么我们广播出去的这两笔交易，最终只会有一笔得到确认。

[![P9GK56.md.png](https://s1.ax1x.com/2018/06/24/P9GK56.md.png)](https://imgchr.com/i/P9GK56)


所以我们改进一下策略，如果我们在自己的服务器上记录nonce，每签名一次交易就增加一次怎么样？
```
var nonce = 0;
async function initializeNonce() {
    nonce = await web3.eth.getTransactionCount(account);
}

function sendTransaction(data) {
    data.nonce = nonce;
    nonce += 1;
    await signAndSend(data);
}
```

在一个分布式网络中，即使不考虑本地出错的可能性，网络传输随时都可能产生错误。设想我们签名好一个交易，并且发送出去，然后增加nonce等待下一次签名。在这个时候，如果刚刚送处去的那个交易失败了怎么办？比如transaction fail或者gas太低直接被系统丢弃了怎么办？如果我们继续增加nonce，由于nonce的不连续会导致后面的交易都得不到处理。

在刚才改进的策略之上，我们的解决方式是不停的监听所有没有被确认的交易，如果超过一定的时间（比如15分钟）交易都没有得到确认。该交易对应的nonce会被重新使用来发送下一个交易。
在网络中查询一个交易是否得到确认可以使用`getTransactionReceipt()`方法。

![solution](http://wx4.sinaimg.cn/mw690/0060lm7Tly1fsn8oxpdlbj30dt0epmy8.jpg)
----
最后，对我们的小玩具感兴趣的，可以用微信扫描二维码来尝试一下：

[![P9GQPK.jpg](https://s1.ax1x.com/2018/06/24/P9GQPK.jpg)](https://imgchr.com/i/P9GQPK)



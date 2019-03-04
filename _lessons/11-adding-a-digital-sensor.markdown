---
layout: post
title:  "添加数字传感器"
permalink: /lessons/adding-a-digital-sensor/
---

## 背景

最近，该公司已将数字传感器纳入自动化部分流程。目标是减少人为错误，提供新的可靠性来源。

仓库操作员仍需检查产品是否符合标准。在仓库操作员发布安全密封请求后，传感器再次执行数字化检查。随后，数字传感器接受或拒绝安全封条请求。

* 如果请求被接受，安全封条将会被发送到产品上；
* 如果请求被拒绝，传感器会向产品发送交易，说明产品未通过检验流程。

![use-case-nem-adding-a-sensor]({{ site.baseurl }}/assets/images/use-case-nem-adding-a-sensor.png)

如果我们将这个应用程序逻辑放在区块链中实现，这个小小的改变意味着抛出仓库操作员已发布的智能合约。这导致花费更多时间进行开发，这会增加平台的成本。


在使用NEM时，我们仍然需要修改我们的代码。在大多数情况下，进行修改就像我们在开发中没有使用区块链技术的应用程序一样。

### 汇总交易

[Aggregated Transactions（汇总交易）](https://nemtech.github.io/concepts/aggregate-transaction.html)将多个事务合并为一个，允许无信任交换和其他高级逻辑。NEM通过生成一次性智能合约来实现这一功能。当所有涉及的帐户都已签署交易时，所有这些帐户都会立即执行汇总交易。

## 实现方法

### 新角色：数字安全传感器

我们应该决定如何在现有项目中表示数字安全传感器。

因为数字传感器需要参与交易，所以它被表示为一个帐户。

创建Digital Safety Sensor(数字安全传感器)账户。

{% highlight bash %}
  nem2-cli account generate --network MIJIN_TEST --url http://localhost:3000 --profile sensor --save
{% endhighlight %}


### 修改createSafetySealTransaction函数

<strong class='tit'>1\. 打开``project/dashboard/src/app/services/safetySeal.service.ts``,修改``transferSafetySeal``函数。</strong>

<strong class='tit'>2\. 创建两个转让交易</strong>

* **operatorToProductTransaction（操作员到产品交易）**:通过产品地址产生转让交易，发送一个company.safety封条。Transfer Transaction to product address, sending one company.safety seal.
* **sensorToProductTransaction（传感器到产品交易）**: 通过产品地址产生转让交易，并通过消息检查。

{% highlight typescript %}
createSafetySealTransaction(productAddress: Address, operatorAccount: PublicAccount): AggregateTransaction {

    const sensorPublicKey = ''; // Todo: Paste sensor account public key
    const sensorPublicAccount = PublicAccount.createFromPublicKey(sensorPublicKey, NetworkType.MIJIN_TEST);

    const operatorToProductTransaction = TransferTransaction.create(
      Deadline.create(),
      productAddress,
      [new Mosaic(new MosaicId('company.safety:seal'), UInt64.fromUint(1))],
      EmptyMessage,
      NetworkType.MIJIN_TEST
    );

    const sensorToProductTransaction = TransferTransaction.create(
      Deadline.create(),
      productAddress,
      [],
      PlainMessage.create('Inspection passed'),
      NetworkType.MIJIN_TEST
    );

    [...]  
}   
{% endhighlight %}

<strong class='tit'>3\. 然后，将这些转让事务包装在汇总事务，表明每笔交易的签名者是谁。</strong>

汇总交易在具备所有必需签名者的签名时完成。

仓库操作员将要签署并发布交易，因此我们还需要数字传感器的签名。我们将这种类型的交易称为``aggregate bonded（汇总绑定）``。

{% highlight typescript %}
createSafetySealTransaction(productAddress: Address, operatorAccount: PublicAccount): AggregateTransaction {

    const sensorPublicKey = ''; // Todo: Paste sensor account public key
    const sensorPublicAccount = PublicAccount.createFromPublicKey(sensorPublicKey, NetworkType.MIJIN_TEST);

    const operatorToProductTransaction = TransferTransaction.create(
      Deadline.create(),
      productAddress,
      [new Mosaic(new MosaicId('company.safety:seal'), UInt64.fromUint(1))],
      EmptyMessage,
      NetworkType.MIJIN_TEST
    );

    const sensorToProductTransaction = TransferTransaction.create(
      Deadline.create(),
      productAddress,
      [],
      PlainMessage.create('Inspection passed'),
      NetworkType.MIJIN_TEST
    );

    return AggregateTransaction.createBonded(
      Deadline.create(),
      [
        operatorToProductTransaction.toAggregate(operatorAccount),
        sensorToProductTransaction.toAggregate(sensorPublicAccount)
      ],
      NetworkType.MIJIN_TEST);
}
{% endhighlight %}

<strong class='tit'>4\. 打开 ``project/dashboard/src/app/components/sendsafetySeal.component.ts``，修改``sendSafetySeal()``函数</strong>

当aggregate transaction(汇总事务)被绑定时，仓库操作员需要至少锁定``10 XEM``。

一旦传感器参与了汇总交易，锁定的XEM数量可以再次在仓库操作员的账户上被使用，并且这两个交易都将自动执行。

<strong class='tit'>5\. 创建一个锁定资金事务，锁定10XEM以获取已签名事务的哈希值。</strong>

{% highlight typescript %}
sendSafetySeal(form) {

    const operatorAccount = Account
      .createFromPrivateKey(form.privateKey, NetworkType.MIJIN_TEST);
    const productPublicKey = Asset.deterministicPublicKey('company', form.selectedProduct);
    const productAddress = PublicAccount.createFromPublicKey(productPublicKey, NetworkType.MIJIN_TEST).address;

    const safetySealTransaction = this.safetySealService.createSafetySealTransaction(productAddress, operatorAccount.publicAccount);
    const signedTransaction = operatorAccount.sign(safetySealTransaction!);

    const lockFundsTransaction = this.safetySealService.createLockFundsTransaction(signedTransaction);

    const signedLockFundsTransaction = operatorAccount.sign(lockFundsTransaction);

    [...]
}
{% endhighlight %}


<strong class='tit'>6\. 发布锁定资金交易并等待确认。然后，发布aggregate bonded（汇总绑定）。</strong>

{% highlight typescript %}
sendSafetySeal(form) {

    const operatorAccount = Account
      .createFromPrivateKey(form.privateKey, NetworkType.MIJIN_TEST);
    const productPublicKey = Asset.deterministicPublicKey('company', form.selectedProduct);
    const productAddress = PublicAccount.createFromPublicKey(productPublicKey, NetworkType.MIJIN_TEST).address;

    const safetySealTransaction = this.safetySealService.createSafetySealTransaction(productAddress, operatorAccount.publicAccount);
    const signedTransaction = operatorAccount.sign(safetySealTransaction!);

    const lockFundsTransaction = this.safetySealService.createLockFundsTransaction(signedTransaction);

    const signedLockFundsTransaction = operatorAccount.sign(lockFundsTransaction);

    this.listener.open().then(ignored => {

      this.startListeners(operatorAccount.address);

      this.transactionHttp
        .announce(signedLockFundsTransaction)
        .subscribe(x => console.log(x), err => console.log(err));

      this.listener
        .confirmed(operatorAccount.address)
        .pipe(
          filter((transaction) => transaction.transactionInfo !== undefined
            && transaction.transactionInfo.hash === signedLockFundsTransaction.hash),
          mergeMap(ignored => this.transactionHttp.announceAggregateBonded(signedTransaction))
        )
        .subscribe(announcedAggregateBonded => console.log(announcedAggregateBonded),
          err => console.log(err));
    });
  }
{% endhighlight %}

### 模拟数字传感器的行为

 ``server``已经实现了模拟数字传感器行为的服务。

打开``project/server/.env``,添加传感器帐户的私钥。

{% highlight bash %}

SENSOR_PRIVATE_KEY='134..............526'

{% endhighlight %}

如果您有兴趣，[在这里查看传感器代码](https://github.com/nemtech/nem2-workshop-nem-applied-to-supply-chain/blob/v0.2.0/project/server/src/service/sensor.service.ts).

    ℹ️ 在生产环境中，您需要执行进一步检查，例如查看发布交易的帐户及其内容。


服务器打开了监听者连接。当新的aggregate bonded transaction(汇总绑定交易)到达此帐户时会通知它。

此时, ``digitalInspection()`` 函数被调用。它返回一个0到5的随机数，如果这个随机数大于2.5，那么这个产品就通过了检查，且交易已经被签署。

如果数字小于2.5，传感器将发送一个带有``Invalid inspection（无效检查）``信息的转让交易。

## 测试修改

<strong class='tit'>1\. **重新启动服务器以应用更改**, 点击``Send Safety Seal``按钮。</strong>

锁定资金交易确认后，您应该在``aggregate bonded transaction(汇总债券交易)``部分下看到该交易。

![screenshot-aggregate-bonded-added]({{ site.baseurl }}/assets/images/screenshot-aggregate-bonded-added.png)

传感器已自动对交易进行了签名或向您的产品发送了转让交易。

<strong class='tit'>2\. 到产品详细信息页面下。现在产品通过安全测试了吗？Go to the product detail page. Has the product passed the safety test?</strong>

![screenshot-product-detail-inspection-failed]({{ site.baseurl }}/assets/images/screenshot-product-detail-inspection-failed.png)

<strong class='tit'>3\. 用不同的产品重复几次，直到其中一个获得安全封条。</strong>

![screenshot-product-detail-inspection-passed]({{ site.baseurl }}/assets/images/screenshot-product-detail-inspection-passed.png)

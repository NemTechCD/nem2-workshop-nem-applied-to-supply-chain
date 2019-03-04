---
layout: post
title:  "注册产品"
permalink: /lessons/registering-products/
---

打开[dashboard（仪表盘）](http://localhost:4200)。仓库操作员使用它来注册产品信息并发送安全封条。但是，与NEM区块链技术的集成并非100%实现。您将通过逐步完成代码来达到让用例运转起来并实现某些功能的目的。

## 背景

注册产品信息包含以下步骤：

1）在已有SQL数据库中创建新产品；

2）创建一个新的区块链account(帐户)；

3）向该帐户发送初始交易。

操作员单击Web app中的``Create Product（创建产品）``按钮。配置了公司帐户的服务器将建立新产品信息，并在区块链上发布交易。

虽然是操作员执行了操作，但我们已经决定从服务器端宣布该事务。这使我们能够在创建时，将公司添加为帐户的所有者。

![use-case-nem-creation]({{ site.baseurl }}/assets/images/use-case-nem-creation.png)

## 实现方法

在``Create Product（创建产品）``标签页下，单击``Create New Product`` 按钮。

![screenshot-register-product]({{ site.baseurl }}/assets/images/screenshot-register-product.png)

    ℹ️服务器，仪表板和catapult bootstrap service都必须处运行状态。

单击该按钮时，您正在向服务器发出HTTP POST请求。该公司的服务器负责创建新产品并将其保存在数据库中。

公钥直接在仪表板中生成。我们把sha256加密哈希算法应用于（'company' +  product ID）。这个新的ID由MySQL数据库生成。

## 产品是否被保存在区块链中？

<strong class='tit'>1\. 单击产品编号。产品信息页展示了产品的状态。如您所见，NEM区块链对此产品一无所知。</strong>

![screenshot-first-transfer-transaction]({{ site.baseurl }}/assets/images/screenshot-first-transfer-transaction.png)

产品将与第一个已发布的交易一起被存储在区块链中。您必须添加一个在区块链中注册产品的函数。

<strong class='tit'>2\. 打开``project/server/.env``，添加这个公司的私钥。</strong>

{% highlight bash %}

COMPANY_PRIVATE_KEY='B...3'

{% endhighlight %}

<strong class='tit'>3\. 打开``project/server/src/controller/product/product.controller.ts``，编辑``createProduct``函数。当产品被保存在数据库后，为产品发送第一笔交易，以便为其创建日期添加时间戳。</strong>

{% highlight typescript %}
export let createProduct: ExpressSignature = (request, response, next) => {
    const productService = new ProductService();

    const companyPrivateKey = process.env.COMPANY_PRIVATE_KEY as string;
    const companyAccount = Account
        .createFromPrivateKey(companyPrivateKey, NetworkType.MIJIN_TEST);

    // Save product in the database and return product created
    return productService.createProduct()
        .pipe(
            mergeMap((product) => {
                return productService.registerProductInBlockchain(companyAccount, product.id).pipe(
                    map((ignored) => product));
            }),
        )
        .subscribe((product) => response.status(200).send(product.toMessage()),
            (err) => response.status(400).send(err));
};
{% endhighlight %}

<strong class='tit'>4\. 打开 ``project/server/src/domain/product/product.service.ts`` ，完成 ``registerProductBlockchain``函数。<strong class='tit'>

{% highlight typescript %}
public registerProductInBlockchain(account: Account, productId: number): Observable<TransactionAnnounceResponse> {

    // Create deterministic public key for product
    const product = Asset.create(account.publicAccount,
        "company",
        productId.toString(),
        {});

    // Create transaction
    const publishableProduct = AssetService.publish(product);

    // Sign transfer transaction with company account
    const signedTransaction = account.sign(publishableProduct);

    return this.transactionHttp.announce(signedTransaction);
}
{% endhighlight %}

我们正在使用第三方库生成第一笔交易[nem2-asset-identifier](https://github.com/aleixmorgadas/nem2-asset-identifier)。

    ℹ️  该库通过以确定的方式创建公钥，帮助我们识别nem区块链中的资产。它还提供了一组标准，用于在代表资产的帐户中进行读写。

创建交易并使用公司帐户签名后，可以向网络进行发布。 **请重新启动服务器以应用更改**。


<strong class='tit'>5\.创建一个新产品,等待交易确认。</strong>

<strong class='tit'>6\.单击产品编号。 根据 **Under Last 10 transactions**标签下，您可以看到在区块链中已注册的产品。</strong>

![screenshot-product-detail]({{ site.baseurl }}/assets/images/screenshot-product-detail.png)

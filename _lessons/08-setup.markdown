---
layout: post
title:  "设置：仓库操作员和安全封条"
permalink: /lessons/setup/
---

创建公司或仓库操作员的账户和马赛克注册是 **只执行一次的操作**。对于这类任务，我们通常使用 **NEM2-CLI**。

## 创建仓库操作员账户

<strong class='tit'>1\.打开命令行，生成一个新帐户。</strong>

{% highlight bash %}
$> nem2-cli account generate
Introduce network type (MIJIN_TEST, MIJIN, MAIN_NET, TEST_NET): MIJIN_TEST
Do you want to save it? [y/n]: y
Introduce NEM 2 Node URL. (Example: http://localhost:3000): http://localhost:3000
Insert profile name (blank means default and it could overwrite the previous profile): operator
{% endhighlight %}

<strong class='tit'>2\. 获取仓库操作员的帐户密钥对和地址。</strong>

{% highlight bash %}
$> nem2-cli profile list
{% endhighlight %}


## 创建公司账户

在区块链中执行操作会产生费用，因为向检验和保护网络的人提供激励是必要的。费用将通过NEM网络的基础加密货币XEM来支付。

    ℹ️ 在私有网络（private network）中，您可以将交易费用设置为0。

现在我们不创建新帐户，而是使用已有XEM的帐户。

<strong class='tit'>1\. 打开命令行（terminal），进入到已下载Catapult Bootstrap Service的目录。</strong>

{% highlight bash %}

$> cd  build/generated-addresses/
$> cat addresses.yaml

{% endhighlight %}

在``nemesis_addresses``下，您将找到包含XEM的密钥对。

<strong class='tit'>2\. 在NEM2-CLI中把第一个帐户作为配置文件加载,此帐户代表了公司。</strong>

{% highlight bash %}
$> nem2-cli profile create

Introduce network type (MIJIN_TEST, MIJIN, MAIN_NET, TEST_NET): MIJIN_TEST
Introduce your private key: 41************************************************************FF
Introduce NEM 2 Node URL. (Example: http://localhost:3000): http://localhost:3000
Insert profile name (blank means default and it could overwrite the previous profile): company

{% endhighlight %}

##  注册公司命名空间

该公司通过发布 [RegisterNamespaceTransaction（注册命名空间事务）](https://nemtech.github.io/guides/namespace/registering-a-namespace.html) 来注册namespace(命名空间)。一旦发布交易，服务器将返回OK响应。

**收到OK响应并不意味着交易有效，因此交易仍然没有被保存在块中**。我们建议在宣布交易有效前密切监控交易状态。

<strong class='tit'>1\. 打开两个新的命令行。第一个命令行监控交易是否发布 **validation errors(验证错误)** 信息。</strong>

{% highlight bash %}
$> nem2-cli monitor status --profile company
{% endhighlight %}

当交易被保存在一个区块中后，您将在 **confirmed** 命令行中看到它。

{% highlight bash %}
$> nem2-cli monitor confirmed --profile company
{% endhighlight %}

<strong class='tit'>2\. 注册namespace（名称空间）``company``，设置块的租赁期限。</strong>

    ℹ️默认情况下，每15秒创建一个blocks(块)。90000个blocks(块)的创建需要大约15,62天。

{% highlight bash %}
$> nem2-cli transaction namespace --name company --rootnamespace --duration 90000 --profile company
{% endhighlight %}

<strong class='tit'>3\. 网络确认交易了吗？检查之前打开的命令行。</strong>

##  注册company.safety子命名空间

完成公司的名称空间的注册后，您就可以创建关联的子命名空间。

注册子命名空间 ``company.safety``:

{% highlight bash %}
$> nem2-cli  transaction namespace --name safety --subnamespace  --parentname company --profile company
{% endhighlight %}

## 创建company.safety:seal马赛克

马赛克名称是``seal``，他的父命名空间``company.safety``使用总计1.000.000``supply``来创建它。

将此马赛克定义为``transferable（可转让）``、``divisibility（可分性）``设置为0（不带小数）、``lease duration(租赁期限)``设置为90000 blocks(块)。``supply mutable（供应可变）``选项允许您在将来增加或减少此类马赛克的数量。

    ℹ️ XEM的divisibility（可分性）为6，1.000在absolute amount（绝对量）中写作1000000000。

<strong class='tit'>1\. 创建马赛克</strong>

{% highlight bash %}
$> nem2-cli transaction mosaic --mosaicname seal --namespacename company.safety --amount 1000000 --transferable --supplymutable --divisibility 0 --duration 90000 --profile company
{% endhighlight %}

<strong class='tit'>2\. 把将1.000 company.safety:seal和1.000 XEM转移到操作员的帐户。</strong>

{% highlight bash %}
$> nem2-cli transaction transfer --profile company
Introduce the recipient address: SA56XXRVS7NG7UH3DTZEMRIVJJLDXXPKAYQAFT2S
Introduce the mosaics in the format namespaceName:mosaicName::absoluteAmount, add multiple mosaics splitting them with a comma:
> company.safety:seal::1000,nem:xem::1000000000

{% endhighlight %}

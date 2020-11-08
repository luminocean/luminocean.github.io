---
title: ' 再谈DNS'
date: "2019-11-08T00:01:07+08:00"
tags:
- network
draft: false
---

DNS本身大家已经很熟悉了，无非就是一个分布式的、用来查询和维护域名与IP地址之间的映射关系的系统。我们这里想再谈一些细节且实用的问题。

### 域名是怎么来的？我可以有自己的域名吗？

当然可以。如果你留意过的话，会发现可以在很多网站上购买域名，而且不同的根域名有着不同的价格。比如在万网上一个`.com`的价格差不多是50多人民币。现在的问题是，当我们在买域名的时候，我们在买些什么？

[根域名的知识](http://www.ruanyifeng.com/blog/2018/05/root-domain.html)里面提到了域名最高管理机构ICANN以及为其管理`.com`和`.net`根域名的公司VeriSign。在任何的域名代理公司（比如万网）购买`.com`和`.net`域名本质上都是向VeriSign申请域名。因此我们在任何地方购买的`.com`和`.net`域名费用，有相当一部分会支付给VeriSign以及ICANN。对于其他域名也有其他的公司负责管理。

那VeriSign到底管理了什么呢？最直观的就是对一个域名的所有权管理，比如一个被注册的域名明显不能再被其他人注册了。其次，还要管理对应的顶级域名服务器。我们熟知全世界有13个[Root name server](https://en.wikipedia.org/wiki/Root_name_server)，但它们的作用仅仅就是记录了所谓[DNS root zone](https://www.iana.org/domains/root/files)的信息。所谓的DNS zone就是能解析一大类域名的DNS服务，比如对于`*.com`或者`*.cloudflare.com`的解析服务，而root DNS zone就是对应各个顶级域名的DNS zone，比如`.com`和`.org`。所以根域名服务器做的就是非常直接的外包，将任何实际的域名解析工作分发到了下级的顶级域名服务器上。它们自己并没有这个世界上所有域名的信息（很明显这个数据量太大也太难维护）。而VeriSign管理的就是`.com`和`.net`的顶级域名服务器（再次注意和根域名服务器的区别），由它们告诉世界上其他的DNS服务器该如何解析`.com`和`.net`域名。

顺带一提，VeriSign和根域名服务器也并不是没有关系。它其实也负责两个根域名服务器的运营（A和J），但这不过是业务的扩展而已。

### A记录与NS记录

那么是不是说`.com`顶级域名服务器就知道如何解析世界上所有`*.com`域名的IP地址呢？这显然不合适也没有必要。每天世界上有着那么多的DNS记录查询和变更，不可能全靠这几台服务器来处理。因此顶级域名服务器们也会或多或少采取类似的外包手段。对于顶级域名本身（比如`google.com`或者`qq.com`），这里的解析会完全由顶级域名服务器来提供，但是具体的提供方式根据需求的不同而会有所区别。

最简单的情况下（比如一个简易的博客站点），我们可能会要求顶级域名服务器把域名直接解析到一个IP地址上，也就是所谓的DNS A记录，然后我们只要在这个IP地址上host我们自己的web service即可。一个实际的例子是Github Pages。如果想自己配置一个Github Pages站点且使用我们自己购买的域名，需要做如下几步：
* 购买域名（废话）
* 在域名供应商网站上设置一条A记录把域名解析到[Github指定的几个IP地址之一](https://help.github.com/en/articles/setting-up-an-apex-domain#configuring-a-records-with-your-dns-provider)。其作用在于：
	* 由我们的供应商通知其管理机构（如果是`.com`域名则管理机构就是VeriSign）在其管辖的DNS服务器上更新我们指定信息（会有一定的生效时间延迟）
	* 生效后，当用户访问域名时会被解析到Github的某一个IP地址上。由于所有的Github Pages用户都在用这几个IP，所以Github实际上用的是virtual host的方法来处理多域名对应单一IP的情况的（也即使用用户浏览器发出请求中的host字段来判断实际访问的是那一个Github Pages service）。根据[Whois Lookup, Domain Availability & IP Search - DomainTools](http://whois.domaintools.com)的查询结果，目前有1,800左右的域名解析到了185.199.110.153这个IP上，由此可估算Github Pages的用户规模。

这种方式简单直接，但是对于很多场景来说是不够的，比如：
* 服务器的IP地址可能会经常修改，但是如果直接依赖于顶级域名服务器，生效时间会比较慢。
* 无法支持次级域名（如mail.example.com）

因此更常见的处理方式是使用NS记录，也即将我们的域名解析到一个DNS服务器上，并且由该DNS服务器负责后续在该域名上的解析。比如我们现在有了一个顶级域名`example.com`，为了在这个域名下使用多级域名（比如`news.example.com`）：
1. 首先搭建一个DNS服务，对外暴露一个public IP，如`w.x.y.z`。可以使用`dnsmasq`之类的工具，并在其配置中设置好各个次级域名的解析。可以使用virtual host之类的技术来减少对于public IP addresses的需求。
2. 在域名代理商网站上（domain’s registrar）为我们的顶级域名`example.com`设置一个NS记录，指向我们的自建DNS服务（authoritative nameserver）。注意，这里是有意思的部分。NS记录并不允许使用IP地址来设置NS记录，原因之一在于使用具体的IP地址会强绑定这个DNS server，如果后期IP地址发生变化会导致DNS服务不可用的问题。另外如果我们的一个域名是由其他公司或部门管理的，那么使用域名而不是具体IP的做法也能提供较好的灵活性。我们决定使用`ns1.example.com`作为我们的ns记录服务器。于是完整的NS记录为`example.com -> ns1.example.com`。这样，如果用户想解析`example.com`，`.com`的DNS会返回`ns1.example.com`指示用户去查询`ns1.example.com`。
3. 更有意思的地方来了，`ns1.example.com`的IP地址是多少？技术上说其实就是我们第一步暴露出来的IP地址`w.x.y.z`。但是对于用户来说，为了知道`ns1.example.com`对应的IP地址，还是会去走`.com` -> `.example.com` -> `ns1.example.com`的路线逐个解析地址。那问题就很明显了，最后问题又变成了`ns1.example.com`的IP是多少，*发生了死循环*。为了解决这个问题，DNS协议引入了`Glue Record`的概念。做法很简单，就是让domain’s registrar除了顶级域名本身的解析，也顺便把这个域名下的nam server的解析也做了，而不要让用户再走正常流程逐层解析。实际操作上，我们只要在domain’s registrar网站上再加一条`ns1.example.com -> w.x.y.z`的A记录就可以了（原来只有`example.com -> ns1.example.com`的NS记录），这就叫glue record。如果使用`dig`命令验证，可以发现这个glue record会在additional section里面出现，作为顶级域名解析请求的附加信息，从而解决这个死循环的问题。



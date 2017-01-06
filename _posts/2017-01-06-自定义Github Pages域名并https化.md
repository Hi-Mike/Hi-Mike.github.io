---
layout: post
title: "自定义Github Pages域名并https化"
date: 2017-01-06
categories: https
---

## 缘由
最近闲逛的时候在万网看到有些域名便宜的可怜。比如我买的 **.win** 域名10年才67RMB，而且第一次买还有优惠券。果断入手了一个。

正好用github搭建的blog有时候还是会放点东西，顺便就解析过去了。

## 域名解析
首先在万网添加CNAME记录，如下图所示，需要解析到的地方自行修改。

![CNAME](http://7xnzl2.com1.z0.glb.clouddn.com/CNAME.png)

然后来到自己创建的Repository下，点击Settings，在里面可以看到Custom domain，如下，添加自己的域名即可。

![domain](http://7xnzl2.com1.z0.glb.clouddn.com/domain.png)

如此，访问 **himike.win** 就可以访问了，如果dns解析较慢，可能要等会儿。

## https化

> 谷歌从 2017 年起，Chrome 浏览器将也会把采用 HTTP 协议的网站标记为「不安全」网站。

背景很重要，当访问网站的时候，不是https的话Chrome直接跳到不是安全链接，这怎么能忍。

github pages本身是https的，然而官方也说的很清楚，自定义域名后是不行的。于是在网上看了下，最简单有效的方法就是使用 [cloudflare](https://www.cloudflare.com) 的免费CDN。

### cloudflare的dns解析
在 [cloudflare](https://www.cloudflare.com) 注册一个帐号，找到SSL服务，填入域名如下，

![domainscan](http://7xnzl2.com1.z0.glb.clouddn.com/domainscan.png)

大约一分钟扫面完后，添加 **DNS解析** ，我这里添加了两条，用 **CNAME** 将自己的域名解析到github博客的地址。

![cnamedns](http://7xnzl2.com1.z0.glb.clouddn.com/cnamedns.png)

然后，还需要把cloudflare的DNS配置到万网，这样我们的域名才会通过CDN进行访问。

![nameservers](http://7xnzl2.com1.z0.glb.clouddn.com/nameservers.png)

### 全局https
配置完上面的步骤，现在当我们访问时

> himike.win => cloudflare dns => cloudflare cdn缓存 => github

cdn访问github的时候仍然是用的http，但是浏览器上是小绿锁了呀。

当然，其实访问 **himike.win** 时候现在还不是小绿锁。一是，dns现在还没刷新，貌似要等的比较久，至少20分钟吧；二是，我们还没有强制使用https。下面看下强制使用https的设置。

![crypto](http://7xnzl2.com1.z0.glb.clouddn.com/crypto.png)

还可以在下面设置下 **Automatic HTTPS Rewrites** ，接下来在Page Rules中添加Page Rule。

![pagerules](http://7xnzl2.com1.z0.glb.clouddn.com/pagerules.png)

貌似在https证书未生效的时候，并没有 **Always Use HTTPS** 选项，相关的选项都没有，生效后还可以添加 **Automatic HTTPS Rewrites** 等设置。

现在访问的时候，所有连接都会强制使用 **https://himike.win** 的域名进行访问。还是但是，当访问带有图片的地方并没有启用https，由于图片使用的七牛的存储并没有使用https，所以会被降级为http。

看了下七牛支持免费SSL，充值10元，绑定域名。暂时这样。
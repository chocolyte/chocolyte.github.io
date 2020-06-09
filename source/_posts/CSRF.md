---
title: CSRF
categories:
  - 技术
tags:
  - 安全
abbrlink: d12d6355
date: 2020-06-09 15:50:22
---

# CSRF 攻击是什么？
CSRF：跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

CSRF能够做的事情包括：以你名义发送邮件，发消息，盗取你的账号，甚至于购买商品，虚拟货币转账......造成的问题包括：个人隐私泄露以及财产安全。例如通过QQ等聊天软件发送的链接(有些还伪装成短域名，用户无法分辨)，攻击者就能迫使Web应用的用户去执行攻击者预设的操作。例如，当用户登录网络银行去查看其存款余额，在他没有退出时，就点击了一个QQ好友发来的链接，那么该用户银行帐户中的资金就有可能被转移到攻击者指定的帐户中。

# CSRF 原理
要完成一个CSRF攻击，必须具备以下几个条件：

* 受害者已经登录到了目标网站（你的网站）并且没有退出

* 受害者有意或者无意的访问了攻击者发布的页面或者链接地址

整个步骤大致是这个样子的：

1. 用户小明在你的网站A上面登录了，A返回了一个session ID（使用cookie存储）

2. 小明的浏览器保持着在A网站的登录状态，事实上几乎所有的网站都是这样做的，一般至少是用户关闭浏览器之前用户的会话是不会结束的

3. 攻击者小强给小明发送了一个链接地址，小明打开了这个地址，查看了网页的内容

4. 小明在打开这个地址的时候，这个页面已经自动的对网站A发送了一个请求，这时候因为A网站没有退出，因此只要请求的地址是A的就会携带A的cookie信息，也就是使用A与小明之间的会话

5. 这时候A网站肯定是不知道这个请求其实是小强伪造的网页上发送的，而是误以为小明就是要这样操作，这样小强就可以随意的更改小明在A上的信息，以小明的身份在A网站上进行操作

整个过程大概如图所示：
![image](https://file.peach.ren/2020/06/csrf-process.png/s)

# 解决方案
![image](https://file.peach.ren/2020/06/csrf-process-1.png/s)

## Synchronizer token pattern
令牌同步模式（Synchronizer token pattern，简称STP）是在用户请求的页面中的所有表单中嵌入一个token，在服务端验证这个token的技术。token可以是任意的内容，但是一定要保证无法被攻击者猜测到或者查询到。攻击者在请求中无法使用正确的token，因此可以判断出未授权的请求。

## Cookie-to-Header Token
对于使用Js作为主要交互技术的网站，将CSRF的token写入到cookie中
```
Set-Cookie: CSRF-token=i8XNjC4b8KVok4uw5RftR38Wgp2BFwql; expires=Thu, 23-Jul-2015 10:25:33 GMT; Max-Age=31449600; Path=/
```

然后使用javascript读取token的值，在发送http请求的时候将其作为请求的header

```
X-CSRF-Token: i8XNjC4b8KVok4uw5RftR38Wgp2BFwql
```

最后服务器验证请求头的 token 是否合法。

## 二次验证
如用户付账输入密码，短信验证码验证等

## 验证码
使用验证码可以杜绝CSRF攻击，但是这种方式要求每个请求都输入一个验证码，显然没有哪个网站愿意使用这种粗暴的方式，用户体验太差，用户会疯掉的。

## 验证 Referer
HTTP头中的Referer字段记录了该 HTTP 请求的来源地址。在通常情况下，访问一个安全受限页面的请求来自于同一个网站，而如果黑客要对其实施 CSRF 攻击，他一般只能在他自己的网站构造请求。因此，可以通过验证 Referer 值来防御 CSRF 攻击。

然而，这种方法并非万无一失。Referer 的值是由浏览器提供的，虽然 HTTP 协议上有明确的要求，但是每个浏览器对于 Referer 的具体实现可能有差别，并不能保证浏览器自身没有安全漏洞。使用验证 Referer 值的方法，就是把安全性都依赖于第三方（即浏览器）来保障，从理论上来讲，这样并不安全。事实上，对于某些浏览器，比如 IE6 或 FF2，目前已经有一些方法可以篡改 Referer 值。如果 bank.example 网站支持 IE6 浏览器，黑客完全可以把用户浏览器的 Referer 值设为以 bank.example 域名开头的地址，这样就可以通过验证，从而进行 CSRF 攻击。

即便是使用最新的浏览器，黑客无法篡改 Referer 值，这种方法仍然有问题。因为 Referer 值会记录下用户的访问来源，有些用户认为这样会侵犯到他们自己的隐私权，特别是有些组织担心 Referer 值会把组织内网中的某些信息泄露到外网中。因此，用户自己可以设置浏览器使其在发送请求时不再提供 Referer。当他们正常访问银行网站时，网站会因为请求没有 Referer 值而认为是 CSRF 攻击，拒绝合法用户的访问。

# Java 项目实践
目前项目中采取的是 Referer 和 Cookie-to-Header 同时验证的方式

示例代码：

```java
// 验证 Referer
String referer = WebUtil.getReferer(request);
if (StringUtil.isEmpty(referer)) {
       throw new RRException(Errors.EMPTY_REFERER);
}

if (!referer.contains(PathUtil.getUrlBase().toString())) {
       throw new RRException(Errors.ERROR_REFERER);
}
```

token 验证可以参考 [shiro github](https://github.com/bdemers/shiro/blob/csrf-and-remember-me-with-JWT/web/src/main/java/org/apache/shiro/web/csrf/StatelessJwtCsrfTokenRepository.java)

# 总结
简单来说，CSRF 其实就是黑客利用浏览器存储用户 Cookie 这一特性，来模拟用户发起一次带有认证信息的请求，比如转账、修改密码等。防护 CSRF 的原理也很简单，在这些请求中，加入一些黑客无法得到的参数信息即可，比如 CSRF Token 或者独立的支付密码等。

# 参考
* [shiro github](https://github.com/bdemers/shiro/blob/csrf-and-remember-me-with-JWT/web/src/main/java/org/apache/shiro/web/csrf/StatelessJwtCsrfTokenRepository.java)
* [shiro github push issue](https://github.com/bdemers/shiro/pull/1/files#diff-0)
* [CSRF 攻击](https://www.jianshu.com/p/b99dc31f1e9f)

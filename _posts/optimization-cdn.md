---
title: 博客优化和CDN加速
abbrlink: cc4a1e83
date: 2018-11-04 11:13:30
categories:
- 运维
tags:
- CDN
- SEO
- HTTPS
---
&#8195;&#8195;内容涉及概述：1. 访问顶级域名自动跳转到www二级域名，在服务端做还是在CDN和域名解析时候做？比如 `abc.com` 跳转到 `https://www.abc.com` ； 2. 配置CDN时候域名解析CNAME冲突？MX冲突？A记录冲突？ 3. CDN分别给顶级域名和二级域名加速，还是只对二级域名加速？ 4. 文章链接更新为 `https://www.abc.com/category/abbrlink.html` 格式。折腾了这么久，网站配置方面的事情应该可以告一段落了。
<!-- more -->

## SEO优化
&#8195;&#8195;我也是第一次接触，涉及皮毛而已，没做title优化、关键字优化、文件压缩、nofollow等，关于运维和优化推荐一个[张戈博客](https://zhangge.net/)，偶然发现，很厉害的样子。我这里其实只涉及一个Hexo永久链接的设置，个人感觉很有必要。

### 文章永久链接
&#8195;&#8195;HTTPS跳转可以后期设置，CDN加速可以以后配置，SEO也可以在必要的时候优化，但是以后再修改文章链接还是挺麻烦的，Hexo默认的文章链接格式是 `abc.com/year/month/day/title/` ，假如一篇 `jstips.md` 的文章，`title` 就是 `jstips` ，这有什么缺点呢？
> 1. 格式不像 `HTML` 静态页面，这个无伤大雅；
> 2. URl层级太多，不利于SEO收录，搜索引擎认为对于一般的中小型站点，3层URL足够承受所有的内容，所以蜘蛛经常抓取的内容是前三层，因此文章的URL最好不要超过2个斜杠；
> 3. 如果文章名字改成了 `jscourse.md` ，文章链接也得变，当然可以在 ` _config.yml`或者文章的 `Front-matter` 设置永久链接。 

&#8195;&#8195;文章链接我们可以使用 `abc.com/title.html` 的格式，但我个人希望看到链接能大概知道这是一篇什么类型的文章，链接中包含文章分类，形如 `abc.com/category/title.html`，其实可以在 ` _config.yml`或者文章的 `Front-matter` 设置永久链接，但是略显麻烦，手动设置永久链接无论如何都不是一个最优解，而且无论英文还是拼音的 `title` 都不够逼格，要用类似 `sba54sb.html` 这样的才够优秀：
>  1. 安装插件，这个插件会根据文章名字生成一个随机的字符串：
```shell
npm install hexo-abbrlink --save
```
> 2. 在 ` _config.yml` 配置：
```
permalink: category/:abbrlink.html
abbrlink:
  alg: crc32  # 算法：crc16(default) and crc32
  rep: hex    # 进制：dec(default) and hex

default_category: uncategorized
category_map:
  运维: operation
  其他: other
tag_map:
```
`category_map` 的功能本来就有，但是没开，我们写文章时候肯定是用中文的分类和标签，但是我们并不希望在链接中出现中文，所以要把中文映射成英文。这样就把 `abc.com/year/month/day/title/` 的链接变为了 `abc.com/category/abbrlink.html` ，像我本文的链接。

### 链接提交
**提交给百度**
&#8195;&#8195;为了让百度收录我们的文章，最开始需要向百度提交链接，主要有两种提交方式：
1. 手动提交；
2. 自动提交，自动提交又包括主动推送、自动推送和sitemap三种方式，推荐主动推送。
> sitemap，定期生成sitemap，然后将sitemap提交给百度。百度会周期性检sitemap然后对其中的链接进行处理，较慢；
> 主动推送，将站点当天新产出**链接立即**通过此方式推送给百度，以保证新链接可以及时被百度收录；
> 自动推送，将自动推送的JS代码部署在站点的每一个页面源代码中，页面在每次**被浏览**时链接会被自动推送给百度。

手动推送直接填写URL就行，sitemap提交也比较比较简单，先生成 `sitemap.xml` 文件，可以用插件简单实现：
```shell
npm install hexo-generator-sitemap --save     
npm install hexo-generator-baidu-sitemap --save
```
&#8195;&#8195;安装完插件后 `Hexo g` 会在 `public` 下面生成 `sitemap.xml` 和 `baidusitemap.xml`，第一个提交谷歌第二个提交百度，我的博客更新量也不大，直接提交了sitemap。
&#8195;&#8195;自动推送的方式，需要修改配置文件 `themes/yelee/layout/_partial/head.ejs` ，添加如下代码：
```js
<script>
(function(){
    var bp = document.createElement('script');
    var curProtocol = window.location.protocol.split(':')[0];
    if (curProtocol === 'https') {
        bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';
    }
    else {
        bp.src = 'http://push.zhanzhang.baidu.com/push.js';
    }
    var s = document.getElementsByTagName("script")[0];
    s.parentNode.insertBefore(bp, s);
})();
</script>
```
**提交给Google**
&#8195;&#8195;把sitemap提交给Goole也很简单，可以参考[Hexo博客收录百度和谷歌](https://www.jianshu.com/p/8c0707ce5da4),这里只附一张图：
![togoogle](/imgs/togoogle.png)

## HTTPS和WWW跳转
### 为什么要跳转
&#8195;&#8195;中午排队打饭时候我一直在纠结：一个几乎没流量的静态小站也需要HTTPS？最后我还是决定了，因为历史的车轮总是滚滚向前，六年前我们将iOS5的系统视若珍宝，认为这是最适合iPhone4S的系统，然而这么多年过去了，iPhone4S都没了别说iOS5。HTTPS也是大势所趋，既然如此，那就折腾吧！
&#8195;&#8195;HTTPS是为了安全，那为什么跳WWW域名？首先一个网站肯定要裸域和WWW都能访问，如果顶级域名不跳到二级域名，意味着 `abc.com` 和 `www.abc.com` 是两个站点，只不过指向了同一个站点，如果你想两个域名都能访问，那么：
> 1. 两个域名都A记录解析到IP最简单，但是致命缺点是真实IP暴露；
> 2. 都做CDN，一来加速二来隐藏真实IP，但是两个域名不利于SEO收录；
> 3. 最好的方式就是访问顶级域名能自动跳转到WWW域名，给WWW域名开CDN。

&#8195;&#8195;实现 `abc.com` 到 `https://www.abc.com` 的跳转可以在服务端配置，也可以靠域名解析配置，因为DNS有URL显性跳转，下面介绍。

### IIS10设置跳转
&#8195;&#8195;我们这里实际上实现了HTTPS和WWW的跳转，证书的申请导入就不说了，导入证书后还需要设置HTTP请求自动跳转到HTTPS请求，直接贴链接了，IIS的配置参考这里【[IIS中实现HTTPS的自动跳转](https://cloud.tencent.com/developer/article/1046824)】，大概步骤：
1. 申请证书；
2. 导入证书；
3. HTTPS绑定，即把证书绑定到对应的网站；
4. 下载URL重写模块，因为IIS没有自带；
5. 配置重写策略，**在你的网站目录下，不是IIS管理面板**；
6. 重启。
配置完后其实就是在根目录下生成一个 `web.config` 
```html
<rule name="http to https" enabled="true" stopProcessing="true">
    <match url="(.*)" />
    <conditions>
	<add input="{HTTP_HOST}" pattern="^(localhost)" negate="true" />
	<add input="{HTTPS}" pattern="^OFF$" />
    </conditions>
    <action type="Redirect" url="https://www.{HTTP_HOST}/{R:1}" redirectType="SeeOther" />
</rule>
```
&#8195;&#8195;这样收到请求 `abc.com` 已经能自动跳转到 `https://www.abc.com` 了，但是 `abc.com` 要想能访问，依然需要一条DNS解析，那解析到IP呢还是解析到CDN呢？我们不是只给WWW域名开CDN吗？这条路貌似走不通？

### DNS设置跳转
&#8195;&#8195;DNS可以设置一个 `URL显性跳转` 的解析，主机记录为 `@` 也就是把裸域 `abc.com` 跳转到记录值 `https://www.abc.com` ，但是如果你之前设置了域名邮箱的解析，那就已经有一条主机记录未为 `@` 的解析也就是从 `abc.com` 解析到 `mxdomain.qq.com.`，这样一来就会产生一个**URL显性跳转和MX的解析冲突**，因为不能同时存在两个主机记录为 `@` 但是记录值是不同域名的的解析。

我们要保留 `URL显性跳转` 就必须修改MX解析：
1. 把MX解析的主机记录换成 `mail` ；
2. 在qq邮箱申请一个 `mail.abc.com` 的域名邮箱，以后用 `admin#mail.abc.com` 收发邮件。

&#8195;&#8195;但实际上，我在执行第二步之前给 `admin#abc.com`（`#`其实是艾特）发了一条邮件，因为我之前申请过一个 `admin#abc.com`，竟然成功收到了？惊喜之余一脸懵逼，因为现在的MX实际上是把 `mail.abc.com` 解析到了`mxdomain.qq.com.`，意味着我们要使用 `admin#mail.abc.com` 来收发邮件啊，结果 `admin#abc.com` 也能成功收发邮件，不知道为什么但却成功了，这是一件很不爽事情，因为你不知道这是不是一个定时炸弹，问了客服也没解决，就先这样吧，补充一下这个邮箱挂过半天，后边又好了，再观察观察吧。一般企业邮箱的MX解析会用 `mail 做主机记录，而且企业邮箱和域名邮箱不是一回事。

### HSTS跳转
&#8195;&#8195;顺便介绍一下HSTS，这种方法目前并未普遍使用。传统的HTTPS跳转，通过一个30x的状态码让浏览器重新发送了一个HTTPS请求，但这样有个缺陷是第一次HTTP请求的建立是明文，可能被劫持，所以有了HSTS。HSTS（HTTP Strict Transport Security），浏览器发起HTTP请求的时候，浏览器将其转换为HTTPS请求，直接略过HTTP请求和重定向，从而使得中间人攻击失效，规避风险。
传统的HTTP到HTTPS流程如图：
![hsts1](/imgs/hsts1.png)
在HTTPS握手之前，是未加密状态，可以被劫持：
![hsts2](/imgs/hsts2.png)
HSTS的流程如下：
![hsts3](/imgs/hsts3.png)
&#8195;&#8195;但是用户首次访问某网站是不受HSTS保护的。因为首次访问时浏览器还未收到HSTS，所以仍有可能通过明文HTTP来访问。解决这个问题有两种方案，一是浏览器预置HSTS域名列表；二是将HSTS信息加入到域名系统记录中。
&#8195;&#8195;实现方法是在服务器返回给浏览器的响应头中，增加 `Strict-Transport-Security` 这个HTTP Header，例如：
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
&#8195;&#8195;就可以告诉浏览器，在接下来的31536000秒内（1年），对于当前域名及其子域名的后续通信应该强制性的只使用HTTPS，直到超过有效期为止。

## CDN
### 介绍
&#8195;&#8195;CDN能**加速**并且**隐藏真实IP**，这就足以让人难以拒绝了，CDN的原理：
访问**未使用CDN**缓存网站的过程：
> 1. 用户在向浏览器输入域名；
> 2. 通过DNS域名解析得到此域名对应的IP地址；
> 3. 浏览器使用所得到的IP地址，向域名的服务主机发出访问请求；
> 4. 浏览器根据域名主机返回的数据显示网页的内容。

访问**使用了CDN**缓存网站的过程：
> 1. 用户在浏览器输入域名；
> 2. 通过DNS域名解析得到此域名对应的CNAME记录，再次对CNAME域名进行解析以得到CDN缓存服务器的IP。在此过程中，使用的全局负载均衡DNS解析，如根据地理位置信息解析对应的IP地址，使得用户访问就近IP；
> 3. 浏览器向缓存服务器发出访问请求；
> 4. 如果缓存服务器上有缓存的数据，则直接返回给用户，否则就需要**回源**，缓存服务器根据浏览器提供的域名，通过Cache内部专用DNS解析得到此域名的实际IP地址，也就是**源站**，再根据**回源host**找到对应的网站提交访问请求；
> 5. 缓存服务器从网站真实IP得得到数据以后，一方面在本地进行保存，以备以后使用，另一方面把获取的数据返回给客户端；

CDN的配置其实也不难，但是要了解几个点：
- 源站，一般是网站服务器的IP；
- 回源host，一般是一个域名，因为一个服务器上可能有多个网站，需要指定具体域名；
- 回源跟随，如果用户请求 `abc.com/1.jpg` ，在节点未命中缓存，则节点会请求源站获取所需资源，若源站返回的状态码为 302跳转的话，如果设置了回源跟随，节点会直接去目标地址请求资源，再把数据返回用户；如果不设置回源跟随，那节点会把302发给用户，让用户去目标地址请求资源，此时如果跳转的域名没有开启CDN，就不会有加速效果。

### CDN解析配置
&#8195;&#8195;在没使用CDN时候，是直接把裸域和www的域名用A记录解析到了真实IP，使用了CDN后的DNS的解析:
> 1. 把直接解析到真实IP的2条A记录暂停或者删除；
> 2. 新增一条CNAME记录，把 `www.abc.com` 解析到CDN节点域名；
> 3. 增加一个DNS，主机记录为 `@` 的URL显性跳转，让 `abc.com` 跳转到 `https://www.abc.com` 即可。

## 参考
[IIS中实现HTTPS的自动跳转](https://cloud.tencent.com/developer/article/1046824)
[HSTS详解](https://www.jianshu.com/p/caa80c7ad45c)
[CDN的实现原理](https://www.cnblogs.com/rayray/p/3553696.html)
[内容分发网络配置预览](https://cloud.tencent.com/document/product/228/6288)
[hexo链接持久化终极解决之道](https://blog.csdn.net/yanzi1225627/article/details/77761488)

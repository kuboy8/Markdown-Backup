---
title: hexo-theme-yelee配置
categories:
  - Web
tags:
  - Hexo
  - Yelee
abbrlink: 39c1ec60
date: 2018-10-18 18:18:02
---

&#8195;&#8195;Hexo有很多简洁美观的主题，这里记录一下hexo-theme-yelee的配置，hexo-theme-yelee是一款Hexo双栏博客主题，配置文件主要有两个：
- 一个`<folder>/_config.yml`，配置网站名字、网站路径和主题目录等；
- 另一个 `<folder>/themes/yelee/_config.yml`，配置主题搜索和评论等。
<!-- more -->
## 配置
&#8195;&#8195;以我的博客为例：
```shell
~$ cd hexo_test
~/hexo_test$ git clone https://github.com/MOxFIVE/hexo-theme-yelee.git themes/yelee
~/hexo_test$ vim _config.yml
```
修改 `theme: landscape`为`theme: yelee`，`hexo s`之后**本地**已经可以访问了。但是存在以下几个问题：
- 主页不显示文章
- 无搜索功能
- 评论系统失效
- 左侧栏GitHub图标为空
- 不蒜子统计失效
- 自定义404
- 跳过指定文件渲染

### 主页显示文章
&#8195;&#8195;刚配置好时候可以在所有文章中看到写的文章，但是在主页空空如也，原因是配置文件的参数和js中不符,查看`themes/yelee/layout/_partial/head.ejs`：
```js
<script>
    var yiliaConfig = {
        fancybox: <%=theme.fancybox%>,
        animate: <%=theme.animate%>,
        isHome: <%=is_home()%>,
        isPost: <%=is_post()%>,
        isArchive: <%=is_archive()%>,
        isTag: <%=is_tag()%>,
        isCategory: <%=is_category()%>,
        fancybox_js: "<%- theme.CDN.fancybox_js %>",
        scrollreveal: "<%- theme.CDN.scrollreveal %>",
        search: <%= theme.search.on %>
    }
</script>
```
&#8195;&#8195;可以看到`search: <%= theme.search.on %>`，而`themes/yelee/_config.yml`里面是:
```
search:
  #onload: ture
  onload: false
```
&#8195;&#8195;修改为：
```
search:
  #onload: ture
  on: false
```
&#8195;&#8195;主页即可显示文章。

### 本地搜索
&#8195;&#8195;需要先安装插件：
```shell
$ npm install hexo-generator-search --save
```
&#8195;&#8195;然后配置文件`themes/yelee/_config.yml`中修改为：
```shell
search:
  on: ture
  #on: false
```
&#8195;&#8195;本地搜索即可使用。

### 评论系统
&#8195;&#8195;第三方评论系统有很多，但是网易云跟帖于2017.8.1停止服务，多说于2017.6.1日停止服务，友言于2018.4.30日关闭服务，Disqus在国内使用不便等等，这里就使用Valine评论系统，Valine基于LeanCloud，所以：
- 首先要注册LeanCloud账号
- 创建应用
- 进入应用查看App ID和APP Key
- 修改_config.yml配置文件
如图：
![create_app](/imgs/create_app.png)

![appid](/imgs/appid.png)
然后修改配置文件：
1. 在`/yelee/_config.yml`中加入：
```
valine:
  on: true
  appid: ***** # App ID
  appkey: ***** # App Key
  verify: true # 验证码
  notify: true # 评论回复邮箱提醒
  avatar: mp # 匿名者头像选项
  placeholder: Just go go!
```
2. 在`CDN`中加入：
```
valine: //unpkg.com/valine@1.2.0-beta1/dist/Valine.min.js
```

3. 在`/yelee/layout/_partial/article.ejs`中加入
```
<% } else if (theme.valine.on){ %>
  <%- partial('comments/valine', {
      key: post.slug,
      title: post.title,
      url: config.url+url_for(post.path)
      }) %>
```
如图：
![art_mod](/imgs/art_mod.png)

4. 创建`/yelee/layout/_partial/comments/valine.ejs`文件，写入：
```js
<section id="comments" style="margin: 2em; padding: 2em; background: rgba(255, 255, 255, 0.5)">
    <div id="vcomment" class="comment"></div>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src="<%- theme.CDN.valine %>"></script>
    <script>
      new Valine({
        el: '#vcomment',
        notify: '<%= theme.valine.notify %>',
        verify: '<%= theme.valine.verify %>',
        app_id: "<%= theme.valine.appid %>",
        app_key: "<%= theme.valine.appkey %>",
        placeholder: "<%= theme.valine.placeholder %>",
        avatar: "<%= theme.valine.avatar %>"
      });
    </script>
</section>
```

5. 在`/yelee/source/css/_partial/mobile.styl`最后加入：
```css
#comments {
    margin: (10/16)rem 10px !important;
    padding: 1rem !important;
}
```

### GitHub图标
&#8195;&#8195;GitHub图标显示异常，查看`themes/yelee/source/css/_partial/customise/social-icon.styl`如下：
```css
.GitHub
    background url(//cdn.bootcss.com/logos/0.2.0/github-octocat.svg) no-repeat white
    background-size 90%
    background-position 50% 100%
```

&#8195;&#8195;可能是链接失效了，那就换成和新浪知乎一样的格式，上面代码直接删除。
1. 下载LOGO
在[iconfont](http://www.iconfont.cn/)搜索“github”，下载LOGO；
![gitlogo](/imgs/gitlogo.png)

2. 修改配置文件：
&#8195;&#8195;图片命名为GitHub.png，然后上传到`themes/yelee/source/imgs/`，在配置文件中img-logo最后一行添加`GitHub white 100`（注意倒数第二行的逗号），100应该是像素（百分百？）大小，可自行调整，如图：
![gitlogocode](/imgs/gitlogocode.png)

### 不蒜子统计
&#8195;&#8195;因七牛域名『dn-lbstatics.qbox.me』于2018年10月初过期，更换为『busuanzi.ibruce.info』，`vim themes/yelee/layout/_partial/after-footer.ejs`:
```js
<script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
```

### 自定义404
**公益404页面**
按照常规操作：
1. `hexo new page 404`
2. `/source/404/index.md`
3. 添加代码，正常我们修改完的代码如下:
```html
title: 404 Not Found：该页无法显示
toc: false
comments: false
permalink: /404
---
<!DOCTYPE html>
<html lang="zh-cn">
<head>
<meta charset="UTF-8" />
<title>404</title>
</head>
<body>
<script type="text/javascript" src="//qzonestyle.gtimg.cn/qzone/hybrid/app/404/search_children.js" homePageName="回到我的主页" homePageUrl="http://www.gokuweb.com"></script>
</body>
</html>
```

试了下会先跳转到我们的404页面，再跳转到寻人页面，而不是直接进入寻人页面，如下：
![custom404](/imgs/custom404.png)

&#8195;&#8195;然后想了想，`source` 下md格式的文章都会被按照主题模板渲染成html，而公益寻人的404实际上不需要做渲染，比如我们的头像，文章分类等等，只需要简单一段JS就足够，所以如果你要展示一个契合自己主题的页面，如上图显示 `404 Not Found：该页无法显示` ，那就用如下代码：
```html
title: 404 Not Found：该页无法显示
toc: false
comments: false
permalink: /404
---
```
如果你想用公益404，那就直接把如下代码覆盖到服务端的 `404.html`：
```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
<meta charset="UTF-8" />
<title>404</title>
</head>
<body>
<script type="text/javascript" src="//qzonestyle.gtimg.cn/qzone/hybrid/app/404/search_children.js" homePageName="回到我的主页" homePageUrl="http://www.gokuweb.com"></script>
</body>
</html>
```
**服务端设置自定义404**
&#8195;&#8195;页面做好后，服务端还要有相应的跳转处理，我在IIS10遇到一个坑，低版本貌似没事，我的 `web.config` 如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <httpErrors errorMode="Custom">
      <remove statusCode="404" subStatusCode="-1" />
      <error statusCode="404" prefixLanguageFilePath="" path="E:\web\404.html" responseMode="File" />
    </httpErrors>
  </system.webServer>
</configuration>
```
虽然IIS是图形界面配置，配置完后实际上会生成一个 `web.config` ，测试时候报错：
```
Absolute physical path "E:\web\404.html" is not allowed in system.webServer/httpErrors section in web.config file. Use relative path instead.
```
这个要修改配置，允许使用绝对路径：
> 1. 左键服务器，双击配置编辑器，不是左键网站，是网站的爷爷目录；
> 2. 搜索 `system.webServer/httpErrors`；
> 3. 把`allowAbsolutePathsWhenDelegated` 的值改为 `True` 。

### 跳过指定文件渲染
&#8195;&#8195;比如 `README.md`、比如百度验证页面、比如谷歌验证页面、比如自定义404页面等，如果我们不想某些文件被渲染，就需要改一下配置文件  `_config.yml` ：
```
skip_render:
    -  README.md
    -  baidu_verify.html
    -  google_verify.html
    -  404/404.html
```
以 `source` 为根目录添加，所以不被渲染的文件实际上是 `source/README.md` 。

### RSS订阅
需要安装插件：
```shell
$ npm install hexo-generator-feed --save
```
&#8195;&#8195;重启服务即可本地查看效果。

## 相关
### Hexo安装
1. 安装Nodjs
```shell
wget https://nodejs.org/dist/v8.12.0/node-v8.12.0-linux-x64.tar.xz
tar xf node-v8.12.0-linux-x64.tar.xz
sudo ln -s /home/zx/node-v8.12.0-linux-x64/bin/node   /usr/local/bin/node
sudo ln -s /home/zx/node-v8.12.0-linux-x64/bin/npm /usr/local/bin/npm
node -v
npm -v
```

2. 安装Git
```shell
sudo apt-get install git
```

3. 安装Hexo
```shell
sudo apt-get install git-core
sudo ln -s /home/zx/node-v8.12.0-linux-x64/lib/node_modules/hexo-cli/bin/hexo /usr/local/bin/hexo
hexo init hexo_test
cd hexo_test
npm install
hexo server
```
&#8195;&#8195;有时候执行hexo提示`hexo: command not found`，是因为环境变量里没有加入hexo，可以通过添加软连接解决。
这样添加软连接是因为环境变量$PATH里面默认有 /usr/local/bin/ 的路径，软链接过来就可以直接使用node、 npm和hexo命令。

### 常用命令：
```shell
$ hexo init hexo_test  #初始化hexo环境，所有博客相关均在该目录下
$ hexo new page about  #创建“关于我”页面
$ hexo new hello-world #创建文章hello-world.md
$ hexo new draft test  #创建草稿test.md，默认不会显示在页面
$ hexo new page tags   #创建标签云页面
$ hexo server          #启动服务器
$ hexo generate        #生成静态文件
$ hexo deploy          #部署网站
$ hexo clean           #清除缓存文件 (db.json) 和已生成的静态文件 (public)
$ hexo list            #列出网站资料
$ hexo migrate <type>  #从其他博客系统[迁移内容](https://hexo.io/zh-cn/docs/migration)。
```

## 参考
[文档](https://hexo.io/zh-cn/docs/)
[简而不减 Hexo 双栏博客主题](https://github.com/MOxFIVE/hexo-theme-yelee)
[Yelee主题使用说明](http://moxfive.coding.me/yelee/)
[Hexo中的Yelee主题，首页不显示文章摘要](https://blog.csdn.net/youshaoduo/article/details/78709160)
[快速开始](https://valine.js.org/quickstart.html)
[评论系统从Disqus到Valine](https://suixinblog.cn/2018/09/valine.html)
[不蒜子](http://ibruce.info/2015/04/04/busuanzi/)
[腾讯寻人](https://xunren.qzone.qq.com/2/1479873225?_t_=0.1637590963522646)
[Custom Error Pages – HTTP Error 500.19 – Internal Server Error](https://blogs.msdn.microsoft.com/benjaminperkins/2012/05/02/custom-error-pages-http-error-500-19-internal-server-error/)
[hexo跳过指定文件的渲染](https://www.qtdebug.com/hexo-skip-render/)

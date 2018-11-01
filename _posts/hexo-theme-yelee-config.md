---
title: hexo-theme-yelee配置
date: 2018-10-18 18:18:02
categories:
- 零碎知识
tags:
- Hexo
- Yelee
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

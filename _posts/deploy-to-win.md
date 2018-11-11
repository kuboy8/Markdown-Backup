---
title: hexo部署到win云服务器
categories:
  - 运维
tags:
  - Windows
  - Hexo
  - Git
  - Docker
  - Web Server
abbrlink: 13b88df5
date: 2018-10-30 00:03:43
---

&#8195;&#8195;我用的Winserver，因为很需要一个win系统所以换Linux是不可能换的。以为很简单的问题折腾了一整天，因为Win无论和Nginx还是Git搭配，都不是那么的行云流水。这里记录一下关于WEB服务器、Docker、Git在Win下**踩的坑**，由入门到精通？不存在的，最终我选择**妥协**！
<!-- more -->

## 部署过程 & 原理
&#8195;&#8195;前前后后工信部备案花了19天，竟然还要公安备案？？？上午9点多收到短信还有点小激动，终于到了showtime  ，接下来就是把本地hexo部署到云服务器，心里默默 `./nginx` 、 配置 `githooks` 、 `hexo clean && hexo g && hexo d`   一顿操作美滋滋，然而进了云服务器后有点懵。

### 过程
> Win+Nginx+Githooks(Win下Nginx性能问题？)——
Win+Docker+Nginx+Githooks(Win下Docker？)——
Win+IIS+Githooks(Win下Git Hooks？)——
Win+IIS+手动Git fetch（妥协）,**不想折腾的朋友已经可以跳过了**，哈哈哈！！

常规hexo部署到linux云服务器方法如下：
**本地操作**
- 安装Nodjs、安装hexo、创建md文章、生成静态html

**云linux**
- 安装Git、初始化裸仓库blog.git、配置Git Hooks，本地执行 `hexo d` 即可。

### hexo部署原理
如图：
![hexo-deploy](/imgs/hexo-deploy.png)
- `hexo g` 把Sources下面的md生成静态html保存到Public；
- `hexo d` 把Public文件夹部署到Git裸仓库；
- `Git Hooks` 自动把仓库文件复制到网站根目录；
- `nginx` 代理用户请求，成功访问html文件。

&#8195;&#8195;其实部署到Win也是这么个流程，但是这里遇到第一个问题：Win+Nginx。

## WEB服务器
&#8195;&#8195;当我解压了Win版本的Nginx时候，忽然想到一个问题，Nginx在Win下性能。

### Nginx
&#8195;&#8195;Nginx是一个高性能的HTTP和反向代理服务器，只能处理静态请求，动态请求转发给相应模块处理然后返回结果。特点是高并发低消耗、处理请求异步非阻塞，可做反向代理、负载均衡，配置简单几乎是静态网站的不二选择，但这个前提是Linux，适用于Windows的nginx版本使用原生Win32 API（而非Cygwin），目前仅使用select()连接处理方法，因此不应期望高性能和可伸缩性【[参考](http://nginx.org/en/docs/windows.html)】。也就是说Win下可以用，但是体现不出它的特点，既然能用其实到这里这部分也可以结束了，但是，我又查了下其他Web服务器。

### IIS
&#8195;&#8195;IIS是集成在Win系统上的，也只能在win下使用，可以说IIS是win系统的御用服务器，只支持静态，开启了“应用程序开发”的一些服务才能支持ASP，当然IIS也支持PHP和JSP，但是常见搭配：IIS+ASP。

### Apache
&#8195;&#8195;Apache本身只支持静态网页，可跨平台支持Win和Linux，Apache区别于Nginx的一点是同步阻塞模型，一个连接对应一个进程，如果要运行JSP的话就需要配合Tomcat，常见搭配：Apache+PHP 、 Apache+Tomcat。
Tomcat其实是一个应用服务器，一个jsp/servlet容器。

### Apache 和 Nginx
&#8195;&#8195;这样一看其实Apache 和 Nginx几乎完全一样，都是本身只支持静态，收到一个PHP请求都要交给PHP去处理，为什么Nginx没有取代Apache？而且经常看到一种说法“Nginx处理静态能力比Apache强，但是Apache处理动态能力比Nginx强，因为Nginx根本没有处理动态能力”，？？？刚才还说Apache只能处理静态的，怎么现在又能处理动态了？虽然Apache 和 Nginx都是把PHP交给相应的模块处理，但是处理方式不同。
> - Nginx是交给PHP-FPM处理。PHP-FPM是一个实现了Fastcgi的程序，扯到CGI就有点懵了，我也只在学习Flask时候略有了解，已经忘光不深入，而 cgi 和 fast-cgi 以独立的进程的形式出现，只要对应的Web服务器实现 cgi 或者 fast-cgi 协议，就能够处理 PHP 请求。
> - Apache是靠mod_php解析PHP。PHP解释器被“内嵌”在Apache的进程里，Apache不会调用任何外部的PHP进程，所以可以说是Apache在处理PHP，这种方式使Apache与PHP能更好的通信。但弊端是哪怕Apache提供的仅仅是静态的资源(如HTML)，Apache的每个子进程也都会载入 mod_php，内存消耗很高。

&#8195;&#8195;相比Nginx+PHP-FPM，Apache+mod_php已经非常成熟，常见搭配：Nginx+Apache+PHP，Nginx做负载均衡，处理静态html，PHP交给Apache。

&#8195;&#8195;既然Win下不能体现Nginx的性能，那我用Docker，Win+Docker+Nginx，这里遇到第二个问题，云服务器下的Docker。

## Docker相关问题
&#8195;&#8195;关于Docker和虚拟机的区别，以及镜像、仓库和容器等就不详细介绍了，参考{% post_link docker-introduce docker学习 %}（先留个接口，有时间把另一篇文章整理下搬过来），这里介绍下我遇到的Docker相关问题。

### Windows/Linux Container
&#8195;&#8195;在Win系统其实是可以安装Docker的，下载[Docker for Windows](https://store.docker.com/editions/community/docker-ce-desktop-windows)，按照文档安装，安装完后提示启用 Hyper-V服务，之后你可能启动失败，报错Hyper-V一个组件无法启动，类似 `“MobyLinuxVM”无法启动` ，正常在本地安装了Hyper-V，在Bios启动Virtualization Technology之后就可以用了，这里应该是受限于云主机的功能，有一个办法可以解决，右键海豚-Swith to windows container，（安装过程中也可以直接选择Windows Container，但这一项默认不勾选），成功启动，总算松了口气，但是总感觉Windows Container怪怪的，果然：
> - Docker on Windows，Windows 10、Windows Server 2016 内置了 Docker，被称为 Docker on Windows，其运行于 Windows NT 内核上，意味着Win自带Docker目前只能运行 Windows应用程序，当前Docker Hub上面的大量镜像无法在Windows Container 中使用，Docker on Windows 极为臃肿，而且启动时间并不快。
> - Docker for Windows，而 Docker 官网下载的，被称为 Docker for Windows。这是我们常说的 Docker，它是运行于 Linux 内核上的 Docker。在 Windows 上运行时实际上是在 Hyper-V 上的一个 Alpine Linux 虚拟机上运行的 Docker，它只可以运行 Linux 程序。

&#8195;&#8195;很不幸Win+Dcker+Nginx之路行不通了。以下是一些其他补充。

### VPS系统内核太低
&#8195;&#8195;Docker要求Linux内核版本必须在要在3.10以上，我们本地安装虚拟机可以为所欲为，但是我用的VPS最高Linux内核才2.6，查了下资料需要升级内核才能使用Docker，折腾了一天铩羽而归，因为我的VPS是OpenVZ结构，OpenVZ结构共用母机内核，所以无法修改。VPS一般有OpenVZ、KVM、Xen这几种。
> - OPenVZ本身运行在linux之上，它通过自己的虚拟化技术把一个服务器虚拟化成多个可以分别安装操作系统的实例，这样的每一个实体就是一个VPS，从客户的角度来看这就是一个虚拟的服务器，可以等同看做一台独立的服务器。**OPENVZ虚拟化出来的VPS只能安装linux操作系统**，而且OpenVZ不是完全的虚拟化，每个VPS账户共用母机内核，**不能单独修改内核**。而这一点也正是其优点，这一共用内核特性使得OpenVZ的效率最高，超过KVM、Xen、VMware等平台。在不超售的情况下OpenVZ是最快速效率最高的VPS平台。
> - KVM、Xen、VMware：这几个VPS平台可以归为一类，它们在虚拟化母机时，是完全的虚拟化，各个VPS示例之间不共用母机内核，各自都是独立 的，几乎所有的操作系统都可以安装到这些被虚拟化出来的VPS上。

### VPS & ECS & CVM
&#8195;&#8195;我们通常说的云服务器，其实包括了阿里的ECS、腾讯的CVM、国外的VPS等，反正不在眼前的都可以这么称呼，这些都有什么区别呢？很尴尬的说，我也不知道！！摘抄一段安慰一下自己：
> - VPS(Virtual Private Server)，中文名称是虚拟专用服务器。详细的定义可以看[维基百科 - VPS](https://zh.wikipedia.org/wiki/%20%E8%99%9A%E6%8B%9F%E4%B8%93%E7%94%A8%E6%9C%8D%E5%8A%A1%E5%99%A8)。通俗点讲就是在一台大型的独立服务器上，通过一定的技术（如虚拟化或者容器化）和一定的软件（VMWare、Xen、KVM、OpenVZ），将这台大型独立服务器的运存、处理器、硬盘进行划分成一个一个小的 VPS，每一台 VPS 都可以分配到独立公网 IP 地址、独立资源和独立系统配置，用户可以安装独立操作系统、单独对自己的 VPS 进行重启和关机。
> - 云主机，常见的CVM(Cloud Virtual Machine)和 ECS(Elastic Compute Service)。理解云主机的概念，就必须抛开一台独立的大型服务器的概念，而要明白一个概念——算池。以阿里云为例。阿里云在国内很多地区都建设了数据中心，在数据中心中所有服务器都是内网互通的。在数据中心里有专门负责存储的机器，配备有大型 HDD 和 SSD 组成 RAID 存储阵列，这些机器组成存储池；有专门负责运算的机器，根据不同的需求有不同的配置（如多核 CPU、强劲的 GPU 和大运存），这些机器组成运算池；有专门进行网络分配和调度的交换机，组成了虚拟网关。

&#8195;&#8195;Win+Dcker+Nginx走不通，只好选择Win+IIS了。

## Win下Git Hooks
&#8195;&#8195;其实一定要用Windows的话，Win+IIS本来也是最优搭配，瞬间回到了原点。IIS还是方便，一路下一步就可以解析静态html了，接下来需要配置Git Hooks。

### 安装Git
&#8195;&#8195;下载Win版本[Git](https://git-scm.com/download/win)，也叫Git for Windows（msysGit），安装之后可以用Git Bash执行Linux下的Git相关命令。

### 配置Git Hooks
1. 初始化裸仓库，裸仓库一般作为服务端的仓库，裸仓库没有工作区，因为服务器上的Git仓库主要是是为了共享：
```shell
mkdir /var/repo
cd /var/repo
git init --bare blog.git
```
2. 配置hookspost-receive文件,钩子有好几种，`git push` 会自动触发 `post-receive`：
```shell
vim blog.git/hookspost-receive
#!/bin/sh
git --work-tree=/var/wwwroot/html --git-dir=/var/repo/blog.git checkout -f
```
3. 修改权限并查看效果：
```shell
chmod +x post-receive
post-receive，./post-receive，source post-receive
```

### 配置本地deploy
&#8195;&#8195;修改_config.yml 文件：
```
deploy:
  type: git
  repo: git@IP:/var/repo/blog.git
  branch: master
  
  type: git
  repo: https://github.com/kuboy8/kuboy8.github.io
  branch: master
```
&#8195;&#8195;成功后就可以实现自动化部署了。

&#8195;&#8195;但是到了这里忽然意识到一个问题，Win系统没有SSH服务。笑容~~逐渐~~瞬间消失。虽然没看过Hexo deploy的源码，但是从配置文件给出的地址格式推测，`Hexo d` 实际上是使用 HTTPS 或者 SSH 协议向Git服务器 git push 数据（如有误欢迎指正），这里我进了一个**误区**，以为要想通过HTTPS协议向Git push数据必须要安装Git服务器，然后搜索了很多资料，已经非常头大了，走马观花看了下，贴个关键字感兴趣的朋友可以继续深入：COPSSH+msysGit、Cygwin、Gogs、Gitlab、Gitblit、Bonobo、Gitstack，实际不需要这么麻烦，我们只想 `hexo d` 能部署到另一台机器Git而已。
 - 想通过SSH协议 push 数据到一个远程仓库，只需要远程服务器支持SSH并且初始化了裸仓库即可。
 - 想通过HTTP/S协议 push 数据到一个远程仓库，主要需要：
> 1. 一个可以支持静态服务的Server，比如Apache；
> 2. 初始化一个裸仓库
> 3. 加载WebDAV

&#8195;&#8195;不过这个参考文档是2006年的，而且用的是Apache，我简单在IIS尝试了下并没有成功，至此已经精疲力尽，无心继续折腾，所以最后选择了手动在远程机器 `git fetch` 或者 `git clone`（HTTPS协议） 。

## 参考
[使用Git Hook自动部署Hexo到个人VPS](https://www.liuxinggang.com/2016-06-17-%E4%BD%BF%E7%94%A8Git-Hook%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2Hexo%E5%88%B0%E4%B8%AA%E4%BA%BAVPS/)
[Windows Container 和 Docker：你需要知道的5件事](https://devopshub.cn/2017/02/03/windows-container-docker-5-things/)
[FAQ列表 - docker中文社区](http://www.docker.org.cn/faq/global/c103.html)
[云主机和 VPS 的区别](https://blog.nfz.moe/archives/compare-vps-ecs-vh.html)
[Git - 在服务器上搭建 Git](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E6%90%AD%E5%BB%BA-Git)
[Git服务器四种协议比较](https://blog.csdn.net/liangpz521/article/details/21534849)
[How to setup Git server over http](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/howto/setup-git-server-over-http.txt)
[Git - Smart HTTP](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Smart-HTTP)

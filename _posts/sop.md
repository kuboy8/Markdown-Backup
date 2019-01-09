---
title: 同源策略和跨域
categories:
  - Web安全
tags:
  - jsonp
  - cors
abbrlink: 28c252a
date: 2019-01-03 17:20:22
---
&#8195;&#8195;同源策略是浏览器最核心也最基本的安全功能，它是由 Netscape 提出的一个著名的安全策略，现在所有支持 JavaScript 的浏览器都会使用这个策略。对应分也有很多实现跨域操作的方式。
<!-- more -->

## 同源策略
### 概念
&#8195;&#8195;同源策略（Same Origin Policy）是浏览器的安全基石，它限制了从同一个源加载的文档或脚本如何与来自另一个源的资源进行交互，简单的说就是 A 网页设置的 Cookie，B 网页不能打开。先看看同源的判断：

|   | URL | 是否同源 | 原因 |
| - | ---------------------------------------- | ----------- | -------- |
| 1 | http://www.example.com/dir/page.html     |             |          |
| A | http://www.example.com/dir2/other.html   | 与 1 同源   |          |
| B | http://example.com/dir/other.html        | 与 1 不同源 | 域名不同 |
| C | http://v2.www.example.com/dir/other.html | 与 1 不同源 | 域名不同 |
| D | http://www.example.com:81/dir/other.html | 与 1 不同源 | 端口不同 |

同源指的是**协议**、**域名**、**端口**都相同。

### 具体影响
&#8195;&#8195;上面说到同源策略控制了不同源之间的交互，例如在使用 `XMLHttpRequest` 或 `<img>` 标签时则会受到同源策略的约束，都有哪些交互呢？通常分三类：
> 1. 通常**允许**跨域**写**操作（Cross-origin writes）。例如链接（links）、重定向、表单提交。特定少数的 HTTP 请求需要添加 preflight。
> 2. 通常**允许**跨域资源**嵌入**（Cross-origin embedding）。包括 `<script>`、`<img>`、`<video>`、` <audio>`、`<frame>/<iframe>`等拥有 `src` 属性的对象，还有 `<link>`。
> 3. 通常**不允许**跨域**读**操作（Cross-origin reads）。包括使用 ajax 获取其他源的数据，以及使用 js 读取其他 iframe 上的 DOM 信息等。但常可以通过内嵌资源来巧妙的进行读取访问。例如可以读取嵌入图片的高度和宽度，调用内嵌脚本的方法，或 [availability of an embedded resource](https://grepular.com/Abusing_HTTP_Status_Codes_to_Expose_Private_Information) 。

&#8195;&#8195;可以看出同源策略主要是限制跨域资源的读取，不影响请求的发送，对于表单提交并不需要获取返回结果，所以不影响；对于 `src` 属性其实是发起了一个 GET 请求，而且对于当前页面来说，存放 JS 文件的域并不重要，重要的是加载 JS 页面所在的域是什么，比如 A 页面用 src 加载了 B 页面的 b.js ，因为它是运行在 A 页面的，所以对于当前 A 页面来说，b.js 的源就是 A 页面而非 B ；对于 Ajax 请求来说，请求虽然可以正常发送到服务器，但是浏览器会阻止 JS 代码拿到服务器的返回值。

&#8195;&#8195;简单的说，不同源的情况下如下行为会受到限制：
> 1. Cookie 无法读取。
> 2. DOM、LocalStorage 和 IndexDB 无法获得。
> 3. AJAX 请求可以发送，但无法获取响应内容。

## 跨域方式

### document.domain
适用范围：
> 1. 两个域只是子域不同。
> 2. 常用于 iframe 窗口与父窗口之间互相获取 Cookie 和 DOM 节点，LocalStorage 和 IndexDB 无法通过这种方法规避同源政策，需要使用 PostMessage API 。

&#8195;&#8195;当两个不同的域只是子域不同时，比如 A 网页是 `http://w1.example.com/a.html`，B 网页是 `http://w2.example.com/b.html`，那么设置相同的 document.domain 即可跨域访问 Cookie：
```
document.domain = 'example.com';
```

&#8195;&#8195;还有一种方法无需设置 document.domain 也可实现跨域 Cookie 访问，服务器也可以在设置 Cookie 的时候，指定 Cookie 的所属域名为一级域名：
```
Set-Cookie: key=value; domain=.example.com; path=/
```
这样的话，二级域名和三级域名不用做任何设置，都可以读取这个 Cookie 。

### Window.name
适用范围：
> 1. 可以是两个完全不同的域。
> 2. 同一个窗口内：即同一个标签页内先后打开的窗口。

&#8195;&#8195;两个网页不同源就无法拿到对方的 DOM。典型的例子是 iframe 窗口和 window.open 方法打开的窗口，它们与父窗口无法通信：
```
document.getElementById("myIFrame").contentWindow.document
//父窗口获取子窗口的 DOM 报错

window.parent.document.body
//子窗口获取主窗口的 DOM 报错
```
&#8195;&#8195;如果两个窗口只是子域名不同，那么设置上一节介绍的 document.domain 属性，就可以规避同源政策，对于完全不同源的网站，目前有三种方法可以解决跨域窗口的通信问题：
> 1. 片段识别符（fragment identifier）
> 2. window.name
> 3. 跨文档通信API（Cross-document messaging）

&#8195;&#8195;浏览器窗口有 window.name 属性。这个属性的最大特点是，无论是否同源，只要在同一个窗口里，前一个网页设置了这个属性，后一个网页可以读取它。window.name 的优点是容量很大，可以放置非常长的字符串；缺点是必须监听子窗口 window.name 属性的变化，影响网页性能。

### window.postMessage
&#8195;&#8195;HTML5 引入了一个全新的API：跨文档通信 API（Cross-document messaging）。这个API为window对象新增了一个 window.postMessage 方法，允许跨窗口通信，不论这两个窗口是否同源：
```
otherWindow.postMessage(message, targetOrigin);
```
otherWindow：接受消息页面的 window 的引用。可以是页面中 iframe 的 contentWindow 属性；window.open 的返回值；通过 name 或下标从 window.frames 取到的值。
message：所要发送的数据，string 类型。
targetOrigin：用于限制 otherWindow，格式“协议 + 域名 + 端口”，* 表示不做限制，向所有窗口发送。

&#8195;&#8195;通过 window.postMessage，还可以读写其他窗口的 LocalStorage 。需要注意的是，postMessage是个非阻塞的调用，也就是说是异步的。Web Messaging 主要用于跨域文档间的通讯，所以它不能用来解决所有跨域调用的问题，例如 ajax 调用。而且IE浏览器对它的支持也很有限。

### JSONP
适用范围：
> 1. 可以是两个完全不同源的域；
> 2. 只支持HTTP请求中的GET方式；
> 3. 老式浏览器全部支持；
> 4. 需要服务端支持。

&#8195;&#8195;它的基本思想是，网页通过添加一个 `<script>` 元素，向服务器请求 JSON 数据，这种做法不受同源政策限制；服务器收到请求后，将数据放在一个指定名字的回调函数里传回来：
```
function addScriptTag(src) {
  var script = document.createElement('script');
  script.setAttribute("type","text/javascript");
  script.src = src;
  document.body.appendChild(script);
}

window.onload = function () {
  addScriptTag('http://example.com/ip?callback=foo');
}

function foo(data) {
  console.log('Your public IP address is: ' + data.ip);
};
```
&#8195;&#8195;通过动态添加 `<script>` 元素，向服务器 example.com 发出请求。注意，该请求的查询字符串有一个 callback 参数，用来指定回调函数的名字，这对于 JSONP 是必需的。服务器收到这个请求以后，会将数据放在回调函数的参数位置返回:
```
foo({
  "ip": "8.8.8.8"
});
```
&#8195;&#8195;由于 `<script>` 元素请求的脚本，直接作为代码运行。这时只要浏览器定义了 foo 函数，该函数就会立即调用。作为参数的 JSON 数据被视为 JavaScript 对象，而不是字符串，因此避免了使用 JSON.parse 的步骤。

&#8195;&#8195;优点：简单适用，老式浏览器全部支持，服务器改造小。不需要 XMLHttpRequest 或 ActiveX 的支持。
&#8195;&#8195;缺点：只支持GET请求。

### WebSocket
&#8195;&#8195;WebSocket 是一种通信协议，使用 `ws://`（非加密）和 `wss://`（加密）作为协议前缀。该协议不实行同源政策，只要服务器支持，就可以通过它进行跨源通信：
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```
&#8195;&#8195;上面代码中，有一个字段是 Origin，表示该请求的请求源，正是因为有了 Origin 这个字段，所以 WebSocket 才没有实行同源政策。因为服务器可以根据这个字段，判断是否许可本次通信。如果该域名在白名单内，服务器就会做出如下回应：
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```
详细参考[WebSocket、Socket、HTTP](https://www.gokuweb.com/reprinted/32927076.html)。

### CORS
适用范围：
> 1. 可以是两个完全不同源的域；
> 2. 支持所有类型的HTTP请求；
> 3. 被绝大多数现代浏览器支持，老式浏览器不支持；
> 4. 需要服务端支持

&#8195;&#8195;CORS（Cross Origin Resource Sharing）跨域资源共享，整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。因此，实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。
&#8195;&#8195;浏览器将 CORS 请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request），只要同时满足以下两大条件，就属于简单请求：
> 1. 请求方法是以下三种方法之一：
> HEAD
> GET
> POST
> 2. HTTP的头信息不超出以下几种字段：
> Accept
> Accept-Language
> Content-Language
> Last-Event-ID
> Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

&#8195;&#8195;凡是不同时满足上面两个条件，就属于非简单请求，对于简单请求，浏览器直接发出 CORS 请求，具体来说就是在头信息之中增加一个 Origin 字段：
```
GET /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```
&#8195;&#8195;上面的头信息中，Origin 字段用来说明，本次请求来自哪个源（协议 + 域名 + 端口）。服务器根据这个值，决定是否同意这次请求。
&#8195;&#8195;如果 Origin 指定的源不在许可范围内，服务器会返回一个正常的 HTTP 回应。浏览器发现这个回应的头信息没有包含 Access-Control-Allow-Origin 字段（详见下文），就知道出错了，从而抛出一个错误，被 XMLHttpRequest 的 onerror 回调函数捕获。注意，这种错误无法通过状态码识别，因为HTTP回应的状态码有可能是200。
&#8195;&#8195;如果Origin指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段：
```
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```
**Access-Control-Allow-Origin**
&#8195;&#8195;该字段是必须的。它的值要么是请求时 Origin 字段的值，要么是一个 * ，表示接受任意域名的请求。

**Access-Control-Allow-Credentials**
&#8195;&#8195;该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。

**Access-Control-Expose-Headers**
&#8195;&#8195;该字段可选。CORS 请求时 XMLHttpRequest 对象的 getResponseHeader() 方法只能拿到 6 个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在 Access-Control-Expose-Headers 里面指定。上面的例子指定，getResponseHeader('FooBar') 可以返回 FooBar字段的值。

详见 [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html) 。

## 引申
### Window 属性
允许以下对 Window 属性的跨源访问：
```
方法
window.blur
window.close
window.focus
window.postMessage

属性	 
window.closed
window.frames
window.length
window.location
window.opener
window.parent
window.sel
window.top
window.window
```

### Location 属性
允许以下对 Location 属性的跨源访问：
```
Methods
location.replace

Attributes	 
URLUtils.href
```

## 参考
[浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
[浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
[我知道的跨域与安全](https://fed.renren.com/2018/01/20/cross-origin/)
[同源策略与JS跨域（JSONP , CORS）](https://segmentfault.com/a/1190000009624849)




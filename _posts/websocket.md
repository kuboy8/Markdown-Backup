---
title: WebSocket、Socket、HTTP
categories:
  - 转载
tags:
  - WebSocket
  - Socket
  - HTTP
abbrlink: '32927076'
date: 2019-01-06 14:39:23
---
&#8195;&#8195;WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。可以实现服务端主动向客户端推送信息，常用于即时通讯，替代轮询，因为轮询会导致过多不必要的请求，浪费流量和服务器资源，每一次请求、应答，都浪费了一定流量在相同的头部信息上。
<!-- more -->

## 轮询
### 短轮询
&#8195;&#8195;短轮询（Polling）方式下，client 每隔一段时间都会向 server 发送 http 请求，服务器收到请求后，将最新的数据发回给 client。一开始必须通过提交表单的形式，这样的后果就是传输很多冗余的数据，浪费了带宽。后来 Ajax 出现，减少了传输数据量。
&#8195;&#8195;在 client 向 server 发送一个请求活动结束后，server 中的数据发生了改变，所以 client 向 server 发送的第二次请求中，server 会将最新的数据返回给 client。
&#8195;&#8195;但这种方式也存在弊端。比如在某个时间段 server 没有更新数据，但 client 仍然每隔一段时间发送请求来询问，所以这段时间内的询问都是无效的，这样浪费了网络带宽。将发送请求的间隔时间加大会缓解这种浪费，但如果 server 更新数据很快时，这样又不能满足数据的实时性。

### Comet
&#8195;&#8195;鉴于（短）轮询的弊端，一种基于 HTTP 长连接的 “服务器推” 的技术被 hack 了出来，这种技术被命名为 Comet。其与（短）轮询主要区别就是，在轮询方式下，要想取得数据，必须首先发送请求，在实时性要求较高的情况下，只能增加向 server 请求的频率；而 Comet 则不同，client 与 server 端保持一个长连接，只有数据发生改变时，server 才主动将数据推送给 client。Comet 又可以被细分为两种实现方式，一种是长轮询机制，一种是流技术。

### 长轮询
&#8195;&#8195;长轮询（Long-polling）方式下，client 向 server 发出请求，server 接收到请求后，server 并不一定立即发送回应给 client，而是看数据是否更新，如果数据已经更新了的话，那就立即将数据返回给 client；但如果数据没有更新，那就把这个请求保持住，等待有新的数据到来时，才将数据返回给 client。
&#8195;&#8195;当然了，如果 server 的数据长时间没有更新，一段时间后，请求便会超时，client 收到超时信息后，再立即发送一个新的请求给 server。
&#8195;&#8195;在长轮询机制下，client 向 server 发送了请求后，server会等数据更新完才会将数据返回，而不是像（短）轮询一样不管数据有没有更新然后立即返回。
&#8195;&#8195;这种方式也有弊端。当 server 向 client 发送数据后，必须等待下一次请求才能将新的数据发送出去，这样 client 接收到新数据的间隔最短时间便是 2 * RTT（往返时间），这样便无法应对 server 端数据更新频率较快的情况。

### 流技术
&#8195;&#8195;流技术基于 Iframe。Iframe 是 HTML 标记，这个标记的 src 属性会保持对指定 server 的长连接请求，server 就可以不断地向 client 返回数据。
&#8195;&#8195;可以看出，流技术与长轮询的区别是长轮询本质上还是一种轮询方式，只不过连接的时间有所增加，想要向 server 获取新的数据，client 只能一遍遍的发送请求；而流技术是一直保持连接，不需要 client 请求，当数据发生改变时，server 自动的将数据发送给 client。

&#8195;&#8195;client 与 server 建立连接之后，便不会断开。当数据发生变化，server 便将数据发送给 client。
&#8195;&#8195;但这种方式有一个明显的不足之处，网页会一直显示未加载完成的状态，虽然我没有强迫症，但这点还是难以忍受。

## WebSocket
&#8195;&#8195;前人推出那么多的解决方案，想要解决的唯一的问题便是怎么让 server 将最新的数据以最快的速度发送给 client。但 HTTP 是个懒惰的协议，server 只有收到请求才会做出回应，否则什么事都不干。因此，为了彻底解决这个 server 主动向 client 发送数据的问题，W3C 在 HTML5 中提供了一种 client 与 server 间进行全双工通讯的网络技术 WebSocket。WebSocket 是一个全新的、独立的协议，基于 TCP 协议，与 HTTP 协议兼容却不会融入 HTTP 协议，仅仅作为 HTML5 的一部分。

&#8195;&#8195;http 协议是一种单向的网络协议，在建立连接后，只有 Browser 向 WebServer 发出请求资源后，WebServer 才能返回相应的数据。但是越来越多的场景需要服务端主动推送的功能，WebSocket protocol 是 HTML5 一种新的协议。它实现了浏览器与服务器全双工通信（full-duplex）。一开始的握手需要借助HTTP请求完成。

Websocket 连接过程：
> 1. 浏览器、服务器建立 TCP 连接，三次握手。这是通信的基础，传输控制层，若失败后续都不执行。
> 2. TCP 连接成功后，浏览器通过 HTTP 协议向服务器传送 WebSocket 支持的版本号等信息。（开始前的 HTTP 握手>）
> 3. 服务器收到客户端的握手请求后，同样采用 HTTP 协议回馈数据。
> 4. 当收到了连接成功的消息后，通过 TCP 通道进行传输通信。

## OSI 模型与 TCP/IP
以下是 [维基百科](http://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B) 中关于OSI 模型的说明：
> 开放式系统互联通信参考模型（英语：Open System Interconnection Reference Model，ISO/IEC 7498-1），简称为OSI模型（OSI model），一种概念模型，由国际标准化组织（ISO）提出，一个试图使各种计算机在世界范围内互连为网络的标准框架。

而 TCP/IP 协议可以看做是对 OSI 模型的一种简化（以下内容来自 [维基百科](http://zh.wikipedia.org/wiki/TCP/IP%E5%8D%8F%E8%AE%AE%E6%97%8F)）：
> 它将软件通信过程抽象化为四个抽象层，采取协议堆叠的方式，分别实作出不同通信协议。协议套组下的各种协议，依其功能不同，被分别归属到这四个阶层之中7，常被视为是简化的七层OSI模型。

这里有一张图详细介绍了 TCP/IP 协议族中的各个协议在 OSI 模型 中的分布：
![TCP/IP 和 OSI 模型](/imgs/OSImodel.png)

&#8195;&#8195;在这里，我们只需要知道，HTTP、WebSocket 等协议都是处于 OSI 模型的最高层： 应用层 。而 IP 协议工作在网络层（第3层），TCP 协议工作在传输层（第4层）。至于 OSI 模型的各个层次都有什么系统和它们对应，这里有篇很好的文章可以满足大家的求知欲：[OSI七层模型详解](http://blog.csdn.net/yaopeng_2005/article/details/7064869) 。

## WebSocket、HTTP 与 TCP
&#8195;&#8195;从上面的图中可以看出，HTTP、WebSocket 等应用层协议，都是基于 TCP 协议来传输数据的。我们可以把这些高级协议理解成对 TCP 的封装。
&#8195;&#8195;既然大家都使用 TCP 协议，那么大家的连接和断开，都要遵循 [TCP 协议中的三次握手和四次挥手](http://blog.csdn.net/whuslei/article/details/6667471) ，只是在连接之后发送的内容不同，或者是断开的时间不同。
更详细内容可阅读：wireshark抓包图解 [TCP三次握手/四次挥手详解](http://www.seanyxie.com/wireshark%E6%8A%93%E5%8C%85%E5%9B%BE%E8%A7%A3-tcp%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B%E8%AF%A6%E8%A7%A3/)。
&#8195;&#8195;对于 WebSocket 来说，它必须依赖 [HTTP 协议进行一次握手](http://tools.ietf.org/html/rfc6455#section-4) ，握手成功后，数据就直接从 TCP 通道传输，与 HTTP 无关了。

WebSocket 和 HTTP 相同点：
> 1. 都是一样基于TCP的，都是可靠性传输协议。
> 2. 都是应用层协议。

WebSocket 和 HTTP 不同点：
> 1. WebSocket是双向通信协议，模拟Socket协议，可以双向发送或接受信息。HTTP是单向的。
> 2. WebSocket是需要握手进行建立连接的。

## Socket 与 WebScoket
&#8195;&#8195;Socket 其实并不是一个协议。它工作在 OSI 模型会话层（第5层），是为了方便大家直接使用更底层协议（一般是 TCP 或 UDP ）而存在的一个抽象层。
&#8195;&#8195;最早的一套 Socket API 是 Berkeley sockets ，采用 C 语言实现。它是 Socket 的事实标准，POSIX sockets 是基于它构建的，多种编程语言都遵循这套 API，在 JAVA、Python 中都能看到这套 API 的影子。
下面摘录一段更容易理解的文字：
> Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

![Socket](/imgs/socket1.png)
![Socket通信](/imgs/sockettraffic.png)

> 主机 A 的应用程序要能和主机 B 的应用程序通信，必须通过 Socket 建立连接，而建立 Socket 连接必须需要底层 TCP/IP 协议来建立 TCP 连接。建立 TCP 连接需要底层 IP 协议来寻址网络中的主机。我们知道网络层使用的 IP 协议可以帮助我们根据 IP 地址来找到目标主机，但是一台主机上可能运行着多个应用程序，如何才能与指定的应用程序通信就要通过 TCP 或 UPD 的地址也就是端口号来指定。这样就可以通过一个 Socket 实例唯一代表一个主机上的一个应用程序的通信链路了。

而 WebSocket 则不同，它是一个完整的 应用层协议，包含一套标准的 API 。
所以，从使用上来说，WebSocket 更易用，而 Socket 更灵活。

## Socket.IO
&#8195;&#8195;Socket.IO 是一个封装了 Websocket、基于 Node 的 JavaScript 框架，包含 client 的 JavaScript 和 server 的 Node。其屏蔽了所有底层细节，让顶层调用非常简单。

&#8195;&#8195;另外，Socket.IO 还有一个非常重要的好处。其不仅支持 WebSocket，还支持许多种轮询机制以及其他实时通信方式，并封装了通用的接口。这些方式包含 Adobe Flash Socket、Ajax 长轮询、Ajax multipart streaming 、持久 Iframe、JSONP 轮询等。换句话说，当 Socket.IO 检测到当前环境不支持 WebSocket 时，能够自动地选择最佳的方式来实现网络的实时通信。

## 参考
[WebSocket（二）-WebSocket、Socket、TCP、HTTP区别](https://www.cnblogs.com/merray/p/7918977.html)
[WebSocket介绍和Socket的区别](https://blog.csdn.net/wwd0501/article/details/54582912)
[WebSocket 与 Socket.IO](https://segmentfault.com/a/1190000006899960)


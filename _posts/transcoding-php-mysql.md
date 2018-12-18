---
title: 浏览器、PHP、MySQL 编码问题
categories:
  - Web安全
tags:
  - 编码转换
abbrlink: ab567a6
date: 2018-12-08 23:10:42
---

&#8195;&#8195;我们输入一个 URL 到浏览器展现出相应的内容，到底经历了什么？今天不谈 ARP缓存、DNS解析、TCP/IP 握手，谈谈浏览器、PHP、MySQL之间请求的**执行流程**以及**编码转换**，编码不统一造成最直接的问题是乱码，还会引发一些安全问题，要合理使用编码函数才能避免这些问题。
<!-- more -->

## 编码介绍
&#8195;&#8195;在计算机的世界里只有 `0` 和 `1` ，但是在人类世界有非常多的语言，最初的 128 个 ASCII 码不足以表达所有的语言，于是各国都扩充了自己的编码，对于一些亚洲国家的文字，一个字节 256 个符号更是远远不够，又相继出现了多字节编码。

### ASCII编码
**单字节编码**
&#8195;&#8195;美国信息互换标准代码（American Standard Code for Information Interchange），使用 7 位二进制数（剩下的1位二进制为0）来表示所有的大写和小写字母，数字、标点符号，以及控制字符，最高位(b7)用作奇偶校验位，用于在代码传送过程中用来检验是否出现错误。0x20 以下的是“控制码”，0x30-0x39 是数字 0-9 ，0x41-0x5A 是大写字母 A-Z ，0x61-0x7a 是小写字母 a-z ，其余是一些符号。

### Unicode字符集
**不是编码**
&#8195;&#8195;不同的编码意味着同一组二进制数可能对应着多种符号，如果有一种编码，将世界上所有的符号都纳入其中，每一个符号都给予一个独一无二的编码，那么乱码问题就会消失。Unicode 应用而生，Unicode 是一个字符集，也叫万国码，它规定了符号的二进制代码，却没有规定这个二进制代码应该如何存储，因为一些汉字可能需要四个字节表示，如果 Unicode 统一规定每个字符四个字节表示，那么每个英文字母都需要用三个字节的 0 填充，这对于存储来说是极大的浪费。

### GBK编码
**双字节编码**
&#8195;&#8195;GBK 编码是 GB2312 的扩充，双字节编码，但是半角状态下的字母是一个字节。GBK 编码总体范围 8140-FEFE，分两部分，第一部分 0x00-0x7F 与 ASCII 码对应。第二部分是汉字编码，第一字节的范围在 0x81-0xFE 之间，第二字节的范围在 0x40-0xFE 之间。
&#8195;&#8195;GB2312 编码总体范围 A1A1-F7FE，第一字节范围是0xA1-0xF7，第二字节范围是0xA1-0xFE。

**用 GBK 解码 UTF-8**
&#8195;&#8195;GBK 解码时，若第一个字节大于 127 就会被认为是 GBK 编码；然而第二个字节不在 GBK 编码范围内时，就无法解析，用字符 `？` 代替错误字节，ASCII码是 63。
&#8195;&#8195;以 `樊` 为例，UTF-8 编码使用三个字节表示该字符 [11100110, 10101000, 10001010]（[E6, A8, 8A]）。使用 GBK 解码时，读到第一个字节大于127，则取两个字节解析为一个 GBK 字符。前两个字节被解析为 `妯`。 第三个字节无法解析，所以是 `？`，最后的结果是 `妯?`。最后一个字节的信息丢失了，由 8A 变成 3F，再使用 UTF-8解析也无法获得正确结果了。
&#8195;&#8195; GBK 和 UTF-8 的转换需要查表，无法直接按照彼此规则计算得出。

### UTF-8编码
**可变字节编码**
&#8195;&#8195;互联网将全世界连在一起，那必然要有一种统一的编码方式，UTF-8 就是在互联网上使用最广的一种 Unicode 的实现方式，规则很简单：
> 1. 对于单字节的符号，字节的第一位设为 0 ，后面 7 位为这个符号的 Unicode 码。因此对于英语字母，UTF-8 编码和 ASCII 码是相同的。
> 2. 对于 n 字节的符号（n > 1），第一个字节的前 n 位都设为 1 ，第 n + 1 位设为 0 ，后面字节的前两位一律设为 10 。剩下的没有提及的二进制位，全部为这个符号的 Unicode 码。

Unicode符号范围与UTF-8二进制对应关系如下：
```
Unicode符号范围        | UTF-8编码方式
(十六进制)             | （二进制）
----------------------+------------------------------------
      0 <--> 0x7f     | 0xxxxxxx
   0x80 <--> 0x7FF    | 110xxxxx 10xxxxxx
  0x800 <--> 0xFFFF   | 1110xxxx 10xxxxxx 10xxxxxx
0x10000 <--> 0x10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```
&#8195;&#8195;比如 `严` 的 Unicode 是 4E25（1001110 00100101），根据上表 4E25 处在 0x800 - 0x10FFFF 范围，因此 `严` 的 UTF-8 编码需要三个字节，格式是 1110xxxx 10xxxxxx 10xxxxxx。然后，从严的最后一个二进制位开始，依次**从后向前**填入格式中的 x ，多出的位补0。这样就得到了 `严` 的 UTF-8 编码是 11100100 10111000 10100101，转换成十六进制就是 E4B8A5 。

**用 UTF-8 解码 GBK**
&#8195;&#8195; UTF-8 解码时候，若解析某个字符失败时使用 `�`（UTF-8 编码为 EF BF BD）代替。以 `樊` 为例，其 GBK 字节码为 [10110111, 10101110]（[B7, AE]），但是该字节码不符合 UTF-8 两字节的编码格式 ，所以两个字节都无法解析，最后的字符串是 `��` 。所有的字节信息都丢失了，因此无法再使用 GBK 解析该字符串。
&#8195;&#8195;注意，UTF-8 用 `�` 替换，是以字符为单位的。例如 [11100110, 10101000, 01000001] 使用 UTF-8 解码得到的结果是 `�A`，而不是 `��A`。根据第一个字节的格式，UTF-8 期望将三个字节转换为一个字符，但最后一个字节不符合要求，所以前两个字节被一个 `�` 代替。而不是每个字节都被 `�` 代替。
&#8195;&#8195; GBK 和 UTF-8 的转换需要查表，无法直接按照彼此规则计算得出。

## 请求处理流程
&#8195;&#8195;我们模拟一个比较主流的架构：Nginx（Apache） + MySQL + PHP ，网站的功能很简单：从 GET 请求获取参数 `id` 的值，然后从 MySQL 查询对应的数据，经浏览器渲染展现给用户，PHP 代码如下：
```
<?php
$con = mysqli_connect('localhost', 'root', '123456') or die('bad!');
mysqli_set_charset($con,'gbk');
mysqli_select_db($con,'test') OR emMsg("连接数据库失败");
$id = isset($_GET['id']) ? mysqli_real_escape_string($con,($_GET['id'])) : 1;
$sql = "SELECT * FROM news WHERE id='{$id}'";
$result = mysqli_query($con,$sql) or die(mysqli_error($con));
?>

<!DOCTYPE html>
<html>
<head>
<meta charset="gbk" />
<title>新闻</title>
</head>
<body>
<?php
$row = mysqli_fetch_array($result,MYSQLI_ASSOC);
echo "<h2>{$row['title']}</h2><p>{$row['content']}<p>\n";
echo '0x'.bin2hex($row['content']) . "<br/>";
echo strlen($row['content']) . "<br/>";
mysqli_free_result($result);
?>
</body>
</html>
```

我们最直观的感受是：
> 1. 浏览器输入URL；
> 2. PHP连接数据库，执行 SQL 查询；
> 3. MySQL 返回结果；
> 4. 浏览器渲染展现。

但实际上还有很多细节：
> 1. 浏览器输入URL；
> 2. 如果请求的是静态资源，Nginx 或者 Apache 会直接处理；如果是动态 PHP 脚本，Nginx 会交给 PHP-FPM 解析，Apache 会调用 mod_php 解析（[Nginx 和 Apache 区别](https://www.gokuweb.com/operation/13b88df5.html#Apache-%E5%92%8C-Nginx)）；
> 3. PHP 引擎解析脚本，以客户端身份连接 MySQL，执行 SQL 查询；
> 4. MySQL 是 C/S 模式，我们都知道 MySQL 有客户端和服务端，服务端收到客户端的查询请求，处理完后把结果返回给 PHP 引擎；
> 5. PHP 处理完脚本呢后，把结果返回 Nginx 或者 Apache；
> 6. Nginx 或者 Apache 把结果返回浏览器，经浏览器渲染展现给用户。

## 编码转换
再往细说：
> 1. 浏览器中输入的参数，是什么编码方式？
> 2. PHP 连接 MySQL 服务端时候是什么编码方式？
> 3. MySQL 服务端执行查询的时候是什么编码方式？
> 4. MySQL 服务端返回的结果是什么编码方式？
> 5. 浏览器展现时候是什么编码方式？

### URL 中编码
&#8195;&#8195;一般来说 URL 只能使用英文字母、阿拉伯数字和某些标点符号，不能使用其他文字和符号，这意味着如果URL中有汉字，就必须编码后使用，我们输入 `text.php?id=你` ，这里的 `你` 会被编码成 `%E4%BD%A0`，用的是 utf-8 编码。
&#8195;&#8195;如果我们输入 `text.php?id=6`，这个 `6` 会被编码成什么呢，0x06 ？ 0110 ？实际上是 ASCII 码，在 ASCII 码中 `6` 对应的是 `0x36`，对应的二进制是 `0011 0110`。
&#8195;&#8195;在我的测试中，不管 URL 路径中的汉字还是参数中的汉字，都是以 UTF-8 的方式编码，和操作系统编码无关，这与阮老师的结论稍有出入，不过可以理解。经过多年的发展，百度和谷歌的网页编码也都是 UTF-8 了，包括微软也在不断进步，随着时间的推移各种标准也肯定会不断地变化。

### PHP 编码
我们首先看看影响页面编码的因素有哪些：
> 1. HTML 标签 `<meta charset="gbk" />`；
> 2. PHP 代码 `header("Content-type: text/html; charset=gb2312");`
> 3. PHP 配置 `default_charset = "UTF-8"`，位于 `php.ini`；

1. 有时候我们明明使用 HTML 标签 `<meta charset="gbk" />` 设置了页面编码，为什么无效呢？从上一章的流程可知，PHP 把数据返回给浏览器，然后浏览器渲染展现给用户，这个标签只是**告诉浏览器用什么编码显示页面**，如果 PHP 返回的数据是 UTF-8 但 `<meta>` 设置了 GBK ，就会产生乱码，可以在网页源代码查看该页面的编码格式。
2. header() 函数向客户端发送原始的 HTTP 报头，该函数**决定了从 PHP 返回的数据是什么编码格式**，而 `<meta>` 标签是高速浏览器用什么编码格式解析这些数据。具体体现在 Response Headers 里面的 Content-Type: charset=UTF-8 字段。
3. PHP 配置其实就是默认添加了 header() 函数，在 PHP 5.6 以后默认配置是 UTF-8 编码，经测试缺省情况下依然是 UTF-8 编码，官方不建议该值为空。
4. 数据展现给用户的过程是 PHP--Web服务--浏览器，所以也可以在 Nginx 或者 Apache 设置编码格式，本质上也是对 Response Headers 做设置，优先级 php header() > php.ini charset > nginx.conf charset。

### MySQL编码
**MySQL 环境变量**
```
character_set_server：默认的内部操作字符集
character_set_client：客户端来源数据使用的字符集
character_set_connection：连接层字符集
character_set_results：查询结果字符集
character_set_database：当前选中数据库的默认字符集
character_set_system：系统元数据(字段名等)字符集
```
以上这些参数如何起作用：

1. 库、表、列字符集的由来
> 1. 建库时，若未明确指定字符集，则采用 character_set_server 指定的字符集。
> 2. 建表时，若未明确指定字符集，则采用当前库所采用的字符集。
> 3. 新增时，修改表字段时，若未明确指定字符集，则采用当前表所采用的字符集。

2. 更新、查询涉及到得字符集变量
> 1. 更新流程字符集转换过程：character_set_client-->character_set_connection-->表字符集。
> 2. 查询流程字符集转换过程：表字符集-->character_set_result

3. character_set_database
> 当前默认数据库的字符集，比如执行 `use xxx` 后，当前数据库变为 `xxx`，若 `xxx` 的字符集为 utf8 ，那么此变量值就变为 utf8(供系统设置，无需人工设置)。

**字符集函数**

&#8195;&#8195;MySQL 是客户端/服务端的设计模式，我们主要关注客户端字符集、连接层字符集、查询结果字符集这三个环境变量，在 PHP 中可以使用函数 `mysql_set_charset` 同时设置以上三个变量为统一的编码，只是**临时设置当前会话**中的这三个环境变量，不影响其他会话。
> 1. 函数 `mysql_set_charset` 在实现了 `set names` 功能之外，还指定了当前字符集编码。关于 `set names` 详细参考[MySQL · 答疑解惑 · set names 都做了什么](https://www.gokuweb.com/reprinted/b3472401.html)。
> 2. `mysql_set_charset` 与 `set names`、`mysql_real_escape_string` 与 `addslashes` 的实例查看[字符集设置](#字符集设置)。

**MySQL中的字符集转换过程**

1. MySQL Server 收到请求时将请求数据从 character_set_client 转换为 character_set_connection；
2. 进行内部操作前将请求数据从character_set_connection转换为内部操作字符集，其确定方法如下：
> • 使用每个数据字段的 CHARACTER SET 设定值；
> • 若上述值不存在，则使用对应数据表的 DEFAULT CHARACTER SET 设定值(MySQL扩展，非SQL标准)；
> • 若上述值不存在，则使用对应数据库的 DEFAULT CHARACTER SET 设定值；
> • 若上述值不存在，则使用 character_set_server 设定值。
3. 将操作结果从内部操作字符集转换为 character_set_results。
![mysqlcharacter](/imgs/mysqlcharacter.png)

**字符集的不统一造成的乱码**
&#8195;&#8195;向默认字符集为 utf8 的数据表**插入** utf8 编码的数据，但是没有设置连接字符集，**查询**时设置了连接字符集为 utf8，那么：
1. 插入时根据 MySQL 服务器的默认设置，character_set_client、character_set_connection 和 character_set_results 均为 latin1。插入操作的数据将经过 latin1--latin1--utf8 的字符集转换过程，这一过程中每个插入的汉字都会从原始的 3 个字节变成 6 个字节保存；
2. 查询时的结果将经过 utf8--utf8 的字符集转换过程，将保存的 6 个字节原封不动返回，产生乱码。
![mysqltans1](/imgs/mysqltrans1.png)

&#8195;&#8195;Latin1 是 ISO-8859-1 的别称，单字节编码，范围是 0x00-0xFF。所以 utf8 编码的 `0xE4BDA0` 被当作三个 Latin1 字符，在向 UTF-8 编码转换时，根据规则 `0xE4` 在 `0x80 -- 0x7FF` 范围，要将其 Unicode 编码 填充到 `110xxxxx 10xxxxxx` 中，得出 `0xC3A4`，以此类推，`0xE4BDA0` 最后转换成 `0xC3A4 C2BD C2A0`。

&#8195;&#8195;向默认字符集为 latin1 的数据表**插入** utf8 编码的数据，提前设置了连接字符集为 utf8：
1. 插入时根据连接字符集设置，character_set_client、character_set_connection 和 character_set_results 均为 utf8。插入数据将经过 utf8--utf8--latin1 的字符集转换，若原始数据中含有 `\u0000~\u00ff` 范围以外的 Unicode 字符，会因为无法在 latin1 字符集中表示而被转换为 `?(0x3F)` 符号。
2. Latin1 是单字节编码，查询时不管连接字符集如何设置都无法正常显示其内容。
![mysqltrans2](/imgs/mysqltrans2.png)

**MySQL在GBK编码下的5C问题**
代码如下：
```
<?php
$link = new mysqli();
$link->real_connect('localhost', 'dztest', 'dztestpasswd', 'enctest',   null, null, MYSQLI_CLIENT_COMPRESS);

#老版本的编码设置，最终数据没有乱码
mysqli_query($link, "SET character_set_connection=gbk,  character_set_results=gbk,character_set_client=binary"); 

$v = "\xd5\x5c\xca\xb5"; // "誠实" 的GBK编码
$v1 = $link->escape_string($v);
echo "$v1\n"; // output: “誠\实"

#新版本编码设置，最终产生乱码
$link->set_charset('gbk'); 
$link->query("SET character_set_client=binary");

#在set_charset后，escape_string会考虑当前字符集GBK.
$v2 = $link->escape_string($v);
echo "$v2\n"; // output: “誠实"

$sql = "INSERT INTO post values(1, '$v1')";
$ret = mysqli_query($link, $sql);
$sql = "INSERT INTO post values(2, '$v2')";
$ret = mysqli_query($link, $sql);
?>
```
&#8195;&#8195;如测试代码中所示，对于带有繁体字的 `誠实`，它的 GBK 编码为 `\xd5\x5c\xca\xb5`，注意到编码中的字节 0x5c 对应的 ASCII 字符就是 `\`。在下面的示例代码中，输出1是 `誠\实`，也就是说 escape_string 只是将 `誠实` 当做普通的 ASCII 字符处理，将 `\xd5\x5c\xca\xb5` 转义成了 `\xd5\x5c\x5c\xca\xb5`，而并不考虑当前的字符集编码为 GBK ，**因为没有设置 escape_string 用到的字符集 mysql->charaset 为 GBK**。恰巧又有 `character_set_client=binary`，于是 mysql 在编码转换的时会进行类似 unescape 处理，最终存储到数据库的是正确的 `誠实`，通过 SELECT hex(content) FROM post，查看发现字段内容为 `d55ccab5`，没有乱码。
&#8195;&#8195;而在新版本的这种设置方式下，输出2是 `誠实`，也就是说在 escape_string 的时候考虑了当前字符集为 GBK ，**因为我们通过 set_charset("GBK") 设置了 escape_string 用到的字符集 mysql->charset=GBK **。转义后还是 `\xd5\x5c\xca\xb5`，而由于 `character_set_client=binary`，在 mysql 中由 `character_set_client->character_set_connection->column character_set` 时，即 `binary->gbk->gbk` 时，会进行 unescape，由于 `\x5c` 后面跟的是并不能 unescape 的字符，最终存储的数据变成了 `帐`，它的 GBK 编码是 `d5ca`。也就是说除了去掉 `\x5c`，还把最后的 `\xb5` 截掉最后留下两个字节。
```
mysql> select * from post;
+------+---------+
| idx  | content |
+------+---------+
|    1 | 誠实  |
|    2 | 帐     |
```

## 引申
### Little/Big endian
&#8195;&#8195;第一节已经提到，UCS-2 格式可以存储 Unicode 码（码点不超过0xFFFF）。以汉字 `严` 为例，Unicode 码是 4E25，需要用两个字节存储，一个字节是 4E ，另一个字节是 25。存储的时候 4E 在前 25 在后，这就是 Big endian 方式；25 在前 4E 在后，这是 Little endian 方式。
&#8195;&#8195;那么很自然的，就会出现一个问题：计算机怎么知道某一个文件到底采用哪一种方式编码？ Unicode 规范定义，每一个文件的最前面分别加入一个表示编码顺序的字符，这个字符的名字叫做"零宽度非换行空格"（zero width no-break space），用 FEFF 表示。这正好是两个字节，而且 FF 比 FE 大 1。如果一个文本文件的头两个字节是 FE FF ，就表示该文件采用大头方式；如果头两个字节是 FF FE ，就表示该文件采用小头方式。

### 绕过 HSTS 抓包

Chrome 浏览器
> 1. 地址栏中输入 chrome://net-internals/#hsts
> 2. 在 Delete domain 中输入项目的域名，并 Delete 删除
> 3. 可以在 Query domain 测试是否删除成功

FireFox 浏览器
> 1. 查找 %APPDATA%\Mozilla\Firefox\Profiles\xxxxxxxx.default\SiteSecurityServiceState.txt
> 2. 删除 SiteSecurityServiceState.txt 文件
> 3. 重启浏览器

Safari 浏览器
> 1. 完全关闭 Safari
> 2. 删除 ~/Library/Cookies/HSTS.plist 这个文件
> 3. 重新打开 Safari 即可
> 4. 极少数情况下，需要重启系统

Opera 浏览器同 Chrome

### 字符集设置
&#8195;&#8195;该函数规定当与数据库服务器进行数据传送时要使用的默认字符集。官方推荐在 PHP>=5.0.5 使用 `mysql_set_charset($con,"gbk");` 来统一数据库操作过程中的编码，作为 `set names 'gbk'` 的升级版，它们之间有什么区别呢？`mysql_set_charset()` 部分源码如下：
```
sprintf(buff, "SET NAMES %s", cs_name);
if (!mysql_real_query(mysql, buff, strlen(buff)))
{
  mysql->charset= cs;
}
```
&#8195;&#8195;可以看到 `mysql_set_charset()` 实现了 `set names` 的功能后（set names 具体功能参考下一节），又对 `mysql->charset=cs` 做了赋值，这个有什么用呢？大家知道 `mysql_real_escape_string()` 不同于 `addslashes` 的地方是会考虑“当前”字符集. 那么这个当前字符集从哪里来呢？就是 `mysql->charset=cs`。
看个实例, 默认 mysql 连接字符集是 Latin-1，（经典的 5c 问题）：
```
<?php
    $db = mysql_connect('localhost:3737', 'root' ,'123456');
    mysql_select_db("test");
    $a = "\x91\x5c";//"慭"的gbk编码, 低字节为5c, 也就是ascii中的"\"
 
    var_dump(addslashes($a));
    var_dump(mysql_real_escape_string($a, $db));
 
    mysql_query("set names gbk");
    var_dump(mysql_real_escape_string($a, $db));
 
    mysql_set_charset("gbk");
    var_dump(mysql_real_escape_string($a, $db));
?>
```
&#8195;&#8195;因为, “慭”的 gbk 编码低字节为 `0x5c` , 也就是 ascii 中的 `\`，而因为除了 `mysql(i)_set_charset` 影响 `mysql->charset` 以外，其他时刻 `mysql->charset` 都为默认值，所以，结果就是：
```
$ php -f 5c.php
string(3) "慭\"
string(3) "慭\"
string(3) "慭\"
string(2) "慭"
```

## 参考
[字符编码笔记：ASCII，Unicode 和 UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
[https://germinate.github.io/2016/GBK与UTF-8编码错误转换后，无法恢复/](GBK与UTF-8编码错误转换后，无法再正确恢复)
[关于URL编码](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html)
[绕过浏览器HSTS限制抓HTTPS数据包](https://mp.weixin.qq.com/s?__biz=MzI0NjYwNTUyNg==&mid=2247483701&idx=1&sn=3a881fffeb52a30f46d26dbfd69707d0&chksm=e9bdf28cdeca7b9a)
[php.ini 核心配置选项说明](http://php.net/manual/zh/ini.core.php#ini.default-charset)
[php乱码相关知识](https://fbd.intelleeegooo.cc/php-encode-file-utf8-chinese-content-type-charset/)
[深入Mysql字符集设置](http://www.laruence.com/2008/01/05/12.html)
[mysql对client发过来的字符处理流程？](https://segmentfault.com/q/1010000000320163)
[深入理解SET NAMES和mysql(i)_set_charset的区别](http://www.laruence.com/2010/04/12/1396.html)
[MySQL在GBK编码下的5C问题](https://www.jianshu.com/p/d6bb69dda00c)

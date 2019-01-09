---
title: MySQL 中 SET NAMES 详解
categories:
  - 转载
tags:
  - MySQL
  - SetNames
abbrlink: b3472401
date: 2018-12-05 18:41:37

---
&#8195;&#8195;在 PHP 中执行 SQL 语句前，通常会设置 set names，相当于同时设置了三个 session 变量，分别是客户端字符集、连接层字符集和结果字符集，这三个字符集对数据库有什么影响呢？
<!-- more -->

## 背景
最近有同事问，`set names` 时会同时设置了3个session变量：
```
SET character_set_client = charset_name;
SET character_set_results = charset_name;
SET character_set_connection = charset_name;
```
&#8195;&#8195;就从变量名字来看，`character_set_client` 是设置客户端相关的字符集，`character_set_results` 是设置返回结果相关的字符集，`character_set_connection` 这个就有点不太明白了，这个有啥用呢？

## 概念说明
通过[官方文档](http://dev.mysql.com/doc/refman/5.6/en/charset-connection.html)来看：
> 1. character_set_client 是指客户端发送过来的语句的编码;
> 2. character_set_connection 是指mysqld收到客户端的语句后，要转换到的编码；
> 3. 而 character_set_results 是指server执行语句后，返回给客户端的数据的编码。

&#8195;&#8195;对人来说，能够理解的是各种各样的符号，而对计算机来说，只能理解二进制，二进制和符号之间的对应关系就是编码。不同地域国家都有自己的一套符号集合，每个都各自用一组二进制数字表示，从而形成了不同的编码，字符集就可以看作是编码和符号的对应关系集合。同一个二进制数在不同的字符集下可能对应完全不一样的字符，如在 GBK 字符集中，`C4E3` 对应的是 `你`，而在 big5 字符集中对应的是 `斕` ，而 `你` 在 unicode 中的编码是 `4F60`，在 [Collation-Charts](http://collation-charts.org/) 这个网站有字符集和编码对应关系图，可以非常直观地看到不同编码下二进制数和符号的对应关系。

&#8195;&#8195;`set names` 设置的3个变量就是设置 mysqld 和客户端通信时，mysqld 应该如何解读 client 发来的字符，以及返回给客户端什么样的编码。

## 实验测试
环境如下：
```mysql
mysql> show variables like 'character%';
+--------------------------+-------------------------------------+
| Variable_name            | Value                               |
+--------------------------+-------------------------------------+
| character_set_client     | utf8                                |
| character_set_connection | utf8                                |
| character_set_database   | utf8                                |
| character_set_filesystem | binary                              |
| character_set_results    | utf8                                |
| character_set_server     | utf8                                |
| character_set_system     | utf8                                |
```
server 端的3个编码设置都是 utf8 。 另外，客户端是标准 mysql client，使用的编码是 utf8 ，和 sever 端编码是一致的。

建一张表作为测试
```mysql
CREATE TABLE t1(id INT, name VARCHAR(200) CHARSET utf8) engine=InnoDB;

INSERT INTO t1 VALUES(0, '你好');
mysql> SELECT id, name, hex(name) FROM t1;
+------+--------+--------------+
| id   | name   | hex(name)    |
+------+--------+--------------+
|    0 | 你好   | E4BDA0E5A5BD |
+------+--------+--------------+
```
下面我们分别改变这3个值，来看下结果会有什么变化。

** Case 1 只改变 character_set_client**
```mysql
SET character_set_client=gbk;
INSERT INTO t1 VALUES(1, '你好');
mysql>  SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id   | name      | hex(name)          |
+------+-----------+--------------------+
|    0 | 你好      | E4BDA0E5A5BD       |
|    1 | 浣犲ソ    | E6B5A3E78AB2E382BD |
+------+-----------+--------------------+
2 rows in set (0.00 sec)
```
可以看到返回的数据已经乱码了，并且数据库里存的确实和第一条记录不一样。

**case 2 只改变 character_set_connection**
```mysql
SET names utf8;
SET character_set_connection = gbk;
INSERT INTO t1 VALUES(2, '你好');

mysql>  SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id   | name      | hex(name)          |
+------+-----------+--------------------+
|    0 | 你好      | E4BDA0E5A5BD       |
|    1 | 浣犲ソ    | E6B5A3E78AB2E382BD |
|    2 | 你好      | E4BDA0E5A5BD       |
+------+-----------+--------------------+
3 rows in set (0.00 sec)
```

**case 3 只改变 character_set_results**
```mysql
SET names utf8;
SET character_set_results = gbk;
INSERT INTO t1 VALUES(3, '你好');

mysql> select id, name, hex(name) from t1;
+------+--------+--------------------+
| id   | name   | hex(name)          |
+------+--------+--------------------+
|    0 |        | E4BDA0E5A5BD       |
|    1 | 你好   | E6B5A3E78AB2E382BD |
|    2 |        | E4BDA0E5A5BD       |
|    3 |        | E4BDA0E5A5BD       |
+------+--------+--------------------+
4 rows in set (0.00 sec)
```
再改回原样，看下结果
```mysql
SET names utf8;
mysql>  SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id   | name      | hex(name)          |
+------+-----------+--------------------+
|    0 | 你好      | E4BDA0E5A5BD       |
|    1 | 浣犲ソ    | E6B5A3E78AB2E382BD |
|    2 | 你好      | E4BDA0E5A5BD       |
|    3 | 你好      | E4BDA0E5A5BD       |
+------+-----------+--------------------+
4 rows in set (0.00 sec)
```

## 分析
我们先理下字符集在整个过程中是怎样变化的，然后再分析上面的case

客户发送请求时：
```
A1 客户端发送出语句(总是以utf8)------> A2 sever收到语句解析(按character_set_client指定编码)
                                                                    |
                                                                    v
A4 数据进入mysqld内部存储<--------- A3 sever判断是否需要转换编码(以character_set_connection 目标编码)
```

server返回结果时：
```
B1 server返回结果(按character_set_results 指定编码) ----->B2客户端解析编码显示(总是以utf8)
```

A3步是否需要转换编码，代码中的逻辑是这样的，在sql_yacc.yy文件中：
```
  LEX_STRING tmp;
  THD *thd= YYTHD;
  const CHARSET_INFO *cs_con= thd->variables.collation_connection;
  const CHARSET_INFO *cs_cli= thd->variables.character_set_client;
  uint repertoire= thd->lex->text_string_is_7bit &&
                   my_charset_is_ascii_based(cs_cli) ?
                   MY_REPERTOIRE_ASCII : MY_REPERTOIRE_UNICODE30;
  if (thd->charset_is_collation_connection ||
      (repertoire == MY_REPERTOIRE_ASCII &&
       my_charset_is_ascii_based(cs_con)))
     tmp= $1;
  else
  {
    if (thd->convert_string(&tmp, cs_con, $1.str, $1.length, cs_cli))
        MYSQL_YYABORT;
  }
  $$= new (thd->mem_root) Item_string(tmp.str, tmp.length, cs_con,
                                      DERIVATION_COERCIBLE,
                                      repertoire);
  if ($$ == NULL)
     MYSQL_YYABORT;
```

&#8195;&#8195;如果 `character_set_client` 和 `character_set_connection` 一样，或者当前的字符编码是和 ASCII 兼容，并且都是 ASCII 范围内的，就不转换，其它情况就转。

&#8195;&#8195;对于 case1 实际上客户端发过来是 UTF8 的，但 A2 步骤 server 认为客户端的编码是 GBK 的，就按 GBK 来解析，同时满足 A3 步骤的转换条件，所以就误将 UTF8 编码认为是 GBK，然后又给转成了 UTF8。 你好的 UTF8 编码是 E4BDA0E5A5BD 6个字节，每个字符3个字节，按 GBK 来解析的话，因为 GBK 是固定2个字节，就认为有3个字符，然后转成 UTF8，虽然 UTF8 是变长的，但是这里的3个 GBK 字符按值都是要占3个字节的，转出来一共9个字节。所以 case1 看到的实际存储的值一共9个字节，比原来的大。 在返回时，是按 UTF8 返回的，因为存了3个 UTF8 字符，所以客户端看到的就是3个。

&#8195;&#8195;对于 case2 A2 步骤没问题，问题是出在 A3，按照转换逻辑，此时需要把 UTF8 转成 GBK，这里因为 `character_set_client` 是正确的，所以转换的源不会识别错，转换成 GBK 自然也不会错，后面存储成 UTF8 时，再从 GBK 转成 UTF8，也没错，因为 UTF8 和 GBK 字符集里都包含 `你` 和 `好` ，所以相互转换也不会出错，只是多了2次转换。

&#8195;&#8195;对于 case3 错在返回字符集设置的和客户端不匹配，在返回时，server 将所有字符转成 GBK 的，结果客户端一根筋的认为是 UTF8 ，就解析错了。 比较有意思的是第二条记录，即 case1 错误插进去的，显示出来是对的。 为什么呢，因为在 case1 中存的时候，是按 `UTF8->强制解析为 GBK->然后转为UTF8` 这个逻辑存下去的，而返回的时候，因为 server 会将存的 UTF8 又给转回 GBK，然后客户端又拿着这个 GBK 误以为是 UTF8 解析，实际上是 case1 的逆向过程，虽然2个方向都是错的，最终显示是好的，所谓的负负得正吧，哈哈。

对于case2 ，数据从客户端进入 server 的时候，多做了2次转换，最终显示还是对的，但不是所有场景都是这样，如下面这种
```mysql
set names utf8;
set character_set_connection  = latin1;
INSERT INTO t1 VALUES(4, '你好');
set names utf8;
mysql>  SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id   | name      | hex(name)          |
+------+-----------+--------------------+
|    0 | 你好      | E4BDA0E5A5BD       |
|    1 | 浣犲ソ    | E6B5A3E78AB2E382BD |
|    2 | 你好      | E4BDA0E5A5BD       |
|    3 | 你好      | E4BDA0E5A5BD       |
|    4 | ??        | 3F3F               |
+------+-----------+--------------------+
5 rows in set (0.00 sec)
```
&#8195;&#8195;为什么呢，因为在 UTF8 转 latin1 时，信息丢失了，latin1 字符编码所能表达的字符集是远小于 utf8 的，`你` 和 `好` 就不在其中，这2个字符在转换中被转成了 `?` 和 `?`，之后存储转换成 UTF8 时，`?` 只有一个字节 `3F` ，还原回去还是 `3F` 。

## 总结
&#8195;&#8195;`character_set_client` 和 `character_set_results` 是一定要和客户端一致，不要依赖于负负得正，`character_set_connection` 设置和 `character_set_client` 不一致，有丢失数据的风险，所以尽量也一致，总之这3个值就是要一样，还要和客户端一致，所以才有了 `set names` 这个快捷命令。关于为啥要有 `character_set_connection` 这一步转换，笔者目前还没看出来，以后理解了再更新，如果读者朋友知道的话，请不吝赐教。

## 参考
[MySQL · 答疑解惑 · set names 都做了什么](http://mysql.taobao.org/monthly/2015/05/07/) 
[mysql中character_set_connection的作用](https://www.cnblogs.com/lazyno/archive/2015/02/07/4278544.html)

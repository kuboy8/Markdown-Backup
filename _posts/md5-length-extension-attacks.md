---
title: MD5长度扩展攻击详解
categories:
  - Web安全
tags:
  - hash
  - MD5
abbrlink: a068ba2f
date: 2018-11-11 15:42:49
---

&#8195;&#8195;MD5长度扩展攻击，简单说就是知道 `MD5(secret + message)` 的**值**和 `secret` 的**长度**前提下，可以推算出 `MD5(secret + message||padding||m’)`，不过 `secret` 长度也可爆破。这里 `padding` 是 `secret` 后的填充字节，有固定填充规则；`m’` 可以是任意数据，`||` 是连接符，可以为空。严格来说也不算是MD5的漏洞，应该说是用法出了问题，比如用 `MD5(message + secret)` 就可避免。
<!-- more -->

&#8195;&#8195;前几天瞟了眼不知道什么年代的一套CMS，忽然发现登录验证用的是 `MD5(passwd + "xxx.com")` 形式，几个字从眼前飘过：MD5长度扩展攻击，就想着整理一篇文章，直到现在才有时间。
&#8195;&#8195;这个漏洞最开始接触其实是在P牛的博客，虽然贴了道哥当年的链接，不过讲真自己看完后还是云里雾里，后来结合i春秋的一篇文章理解了差不多，不过代码稍有残缺，原理方面和其他资料差不多，**重点在代码部分**。其实长度扩展攻击并不局限于MD5，基于[Merkle–Damgård](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction)构造的算法基本都存在这个问题。

## 原理
&#8195;&#8195;先抄个图：
![md5flowchart ](/imgs/md5flowchart.png)
&#8195;&#8195;根据这个图我们知道MD5算法流程如下：
> 1. 把消息以512位长度为单位分为n个消息块；
> 2. 最后一组必然不足512位，对最后一个消息块进行消息填充（padding），填充完后其长度为是512位；
> 3. 每个消息块会和一个输入量做运算，把运算结果作为下一次运算的输入量；
> 4. 最后一个运算结果分四组，每组逆序得出最终hash值。

&#8195;&#8195;假如原消息是 `secret + message`，长度为612位，处理时候需要分成2个消息块，第2个消息块100位，需要填充412位，中间具体怎么计算不得而知，但是我们知道最后是算出一个 `hash值`，这个 `hash` 值本来是作为下一次运算的输入量，但因为已经到了最后一步，所以这个 `hash` 值被分成四组，然后每组逆序得出了最终的MD5值：就是 `MD5(secret + message)` ，如果我们能让这个 `hash值` 继续计算下去呢？让它作为下一个运算的输入量继续下去而非分组取逆结束。所以我们要做的就是随便增加点数据，消息长度不满足512位整数倍后让第3个消息块的计算进行下去，而第3个消息块的计算是我们可控的，所以我们可以得到 `MD5(secret + message||padding||m’)` 的值。
&#8195;&#8195;MD5算法可以概括为三步：
> 1. 分组；
> 2. 填充，也叫Padding；
> 3. 复杂计算，依次对每组消息进行压缩，每组经过64轮的数学变换。不过最开始的四组初始化向量是硬编码写死的，不是随机数。

主要介绍下Padding，这是需要我们构造的。

## Padding过程
&#8195;&#8195;MD5算法会对消息进行分组，每组512 bit（64 byte），不足512 bit 的部分会用padding补位。补位的数据由三部分组成（连原始消息总共四部分，原消息长度用 L0 表示）：
> 1. 第一部分**长度固定**1字节，`L1 = 1` ，填充**内容固定**，16进制下为 `0x80` ；
> 2. 第二部分填充**内容固定** `0x00` ，长度由其他三部分决定，16进制下为 `L2 = 64-L0-L1-L3` 字节；
> 3. 第三部分**长度固定**8字节，`L3 = 8` ，填充内容为补位前消息的长度，小端存储。

&#8195;&#8195;比如对消息 `message` 填充，消息长度7字节，所以 `L0 = 7`，`L1 = 1` 和 `L3 = 8` 是固定的，可以计算 `L2 = 64-7-1-8 = 48 byte`，所以第一部分是长度1字节的 `0x80` ，第二部分是长度48字节的 `0x00` ，第三部分是长度8字节的 `0x38` ，因为7字节56位 。填充完后就是 6D 65 73 73 61 67 65 80 00 00 … 00 38 00 00 00 00 00 00 00 。
如图：
![padding](/imgs/padding.png)

## 具体操作
&#8195;&#8195;就以CTF的题目为例，实际应用中可举一反三，不仅仅局限于登录验证：
```html
<html>
<body>
<?php
echo 'username:'.$_POST["username"].'<br>';
echo 'password:'.$_POST["password"].'<br>';
$flag = "XXXXXXXXXXXXXXXXXXXXXXX";
$secret = "udontkownthenum"; // This secret is 15 characters long for security!
$username = $_POST["username"];
$password = $_POST["password"];
if (!empty($_COOKIE["getmein"])) {
    if (urldecode($username) === "admin" && urldecode($password) != "admin") {# ===俩边不管值还是类型都要一致
        if ($_COOKIE["getmein"] === md5($secret . urldecode($username . $password))) {
            echo "Congratulations! You are a registered user.\n";
            die ("The flag is ". $flag);#exit()/die() 函数输出一条消息，并退出当前脚本
        }
        else {
            die ("Your cookies don't match up! STOP HACKING THIS SITE.");
        }
    }
    else {
        die ("You are not an admin! LEAVE.");
    }
}
setcookie("sample-hash", md5($secret . urldecode("admin" . "admin")), time() + (60 * 60 * 24 * 7));
if (empty($_COOKIE["source"])) {
    setcookie("source", 0, time() + (60 * 60 * 24 * 7));
}
else {
    if ($_COOKIE["source"] != 0) {
        echo "xiaolaodi"; // This source code is outputted here
    }
}
?>
<h1>Admins Only!</h1>
<p>If you have the correct credentials, log in below. If not, please LEAVE.</p>
<form method="POST" action="">
    Username: <input type="text" name="username"> <br>
    Password: <input type="password" name="password"> <br>
    <button type="submit">Submit</button>
</form>
</body>
</html
```
我们需要构造的cookie如下：
```html
if ($_COOKIE["getmein"] === md5($secret . urldecode($username . $password)))
```
### 计算输入量
&#8195;&#8195;整理下思路，首次运算时第一个消息块和四组初始化向量经过复杂运算，得出一个hash值，它会作为下一次运算的输入量，和第二个消息块进行复杂运算，以此类推，所以我们要：
> 1. 根据最终MD5值逆向计算出hash值，这个值要作为输入量；
> 2. 根据padding规则计算出填充好的消息块；
> 3. 输入量和hash值作为参数带入下一轮复杂运算，我们不需要知道具体的复杂运算。

&#8195;&#8195;题目中格式是 `md5(secert + adminadmin)` ，我们要把它逆向回高低位互换之前的状态，然后将其作为下一次运算的输入量，这个值其实手算也很简单，比如我们获取到的MD5值是 `3e9ac7a5668377a8c63346fe90c0517f` ，分组逆序后：
```
0x3e9ac7a5 —— 0xa5c79a3e
0x668377a8 —— 0xa8778366
0xc63346fe —— 0xfe4633c6
0x90c0517f —— 0x7f51c090
```
![ctf-hlea](/imgs/ctf-hlea.png)

&#8195;&#8195;但手算是不可能手算的，道哥用的是JS实现的，涉及左移运算 `<<` 、有符号右移运算 `>>` 、无符号右移运算 `>>>` ，左移运算不分有无符号，是一定保留符号位的。`byte` 转换 `int` 的时候还用了 `&0xff` ，因为 `byte` 转换为 `int` 类型的时候会高位补1，这样二进制补码就会变化，`&0xff` 就是为了保持转换过程中的低八位一致。我们这里用python实现一下：
```python
# -*- coding: UTF-8 -*-
def rever_hash():
    samplehash="3e9ac7a5668377a8c63346fe90c0517f"
    s = [samplehash[i:i+8] for i in range(0,len(samplehash),8)]
    a = []
    for i in range(len(s)):
        print "分组{}:{}".format(i+1, s[i])
        a.append(s[i].decode('hex')[::-1].encode('hex'))
        print "s{} =  {}".format(i+1, a[i])
        print "-----------"
    print "==============="
    print a
    s1 = int(a[0],16)
    s2 = int(a[1],16)
    s3 = int(a[2],16)
    s4 = int(a[3],16)
    print s1,s2,s3,s4

if __name__ =='__main__':	
    rever_hash()
```
&#8195;&#8195;还原出四组初始向量，或者说输入量。这里 `hex函数` 能转换为16进制，但是参数必须是10进制的值，且转换完的16进制是字符串形式，不能拿来直接用，int转换后的结果是10进制形式显示，没有16进制看着直观。所以我这里用字符串的形式打印s1,s2,s3,s4方便查看对比，用10进制形式对s1,s2,s3,s4进行赋值，其实就算定义一个值为16进制，比如a=0xa5c79a3e，打印出来也是10进制形式，都是int整形。
&#8195;&#8195;如果要对一个10进制的数按字节逆序，比如i = 1050331045要用下面的代码：
```
hex(i)[2::].decode('hex')[::-1].encode('hex')
```
其实是做了三件事情：
> 1. 使用hex()得到字符串格式的16进制数0x3e9ac7a5，然后把开头的0x去掉得到str格式的3e9ac7a5；
> 2. 使用字符串解码deocde，按十六进制解码，不经过这一步，逆转的时候是以一个字母一字节，结果5a7ca9e3，但是经过16进制解码，16进制是2个数字一字节（1个数字对应2进制4位），现在按一个字节逆转的结果a5c79a3e；
> 3. 使用字符串编码encode，编码成十六进制，得到结果0xa5c79a3e。

### 填充消息块
&#8195;&#8195;题目中给出了 `secert` 长度为15，我们不知道具体值，假设是15个A，我们输入的是 `adminadmin` ，根据Padding的规则，其实我们手动构造也不难：
![padding2](/imgs/padding2.png)
代码实现：
```python
# -*- coding: UTF-8 -*-
import my_md5

def padding():
    secret = "A"*15
    secret_admin = secret + 'adminadmin{padding}'
    padding = '\x80{zero}\xc8\x00\x00\x00\x00\x00\x00\x00'.format(zero="\x00"*(64-15-10-1-8))
    print "padding:",(padding,16)
    secret_admin = secret_admin.format(padding=padding) + 'dawn'
    print "secret_admin:",secret_admin
    r = my_md5.deal_rawInputMsg(secret_admin)
    inp = r[len(r)/2:]
    
    md5 = my_md5.run_md5(s1,s2,s3,s4,inp)
    print "getmein:", md5

if __name__ =='__main__':	
    padding()
    md5 = my_md5.run_md5(s1,s2,s3,s4,inp)
    print "getmein:", md5

```
&#8195;&#8195;这里引入了 `my_md5`，这个文件代码有点多，放到最后，我也添加了简单的注释，主要包括了对消息块的填充和64轮一循环的复杂运算。我们上面的填充是针对原消息 `secertadminadmin` 的，后面调用函数实现了对添加 `dawn` 之后消息块的填充。最后MD5的计算只有两行，我也写进了上面函数，下面解释。

### 计算MD5值
&#8195;&#8195;计算出输入量和填充好的消息块，就可以直接带入函数了，消息经过两次填充后达到了1024位，相当于我们劫持了第一次运算的结果，构造了第二次运算所需的参数，所以这里的消息块取后512位即可：
```
inp = r[len(r)/2:]
```
代码自己调整一下就行了，全写一个函数里也行，我在代码里加了命令参数，主要是想多实现点功能，不过先就这样吧，详细代码就不贴了，篇幅太长，运行如下：
![get-md5](/imgs/get-md5.png)
修改post值和cookie值：
![modify-request](/imgs/modify-request.png)
获取flag：
![get-flag](/imgs/get-flag.png)

附 my_md5.py
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys

#‘>Q’：整形(8字节)大端保存, 然后做16进制编码
def genMsgLengthDescriptor(msg_bitsLenth):

    '''
    ---args:
            msg_bitsLenth : the bits length of raw message

    --return:

    '''
    #print "genMsgLengthDescriptor:", __import__("struct").pack(">Q",msg_bitsLenth).encode("hex")
    return __import__("struct").pack(">Q",msg_bitsLenth).encode("hex")

#‘<Q’：整形(8字节)小端保存, 然后做16进制编码
def reverse_hex_8bytes(hex_str):

    '''
    --args:

            hex_str: a hex-encoded string with length 16 , i.e.8bytes

    --return:

            transform raw message descriptor to little-endian 

    '''

    hex_str = "%016x"%int(hex_str,16)

    assert len(hex_str)==16
    #print "reverse_hex_8bytes:", __import__("struct").pack("<Q",int(hex_str,16)).encode("hex")
    return __import__("struct").pack("<Q",int(hex_str,16)).encode("hex")

#‘<L’：整形(4字节)小端保存, 然后做16进制编码
def reverse_hex_4bytes(hex_str):

        '''

        --args:

                hex_str: a hex-encoded string with length 8 , i.e.4bytes

        --return:

                transform 4 bytes message block to little-endian

        '''    

        hex_str = "%08x"%int(hex_str,16)

        assert len(hex_str)==8    

        return __import__("struct").pack("<L",int(hex_str,16)).encode("hex")

# 对传入的参数做填充.
def deal_rawInputMsg(input_msg):

    '''

    --args:

            input_msg : inputed a ascii-encoded string

    --return:


    '''

    ascii_list = [x.encode("hex") for x in input_msg]
    #print "list",ascii_list
    length_msg_bytes = len(ascii_list)

    length_msg_bits = len(ascii_list)*8

    #padding
    i=0
    ascii_list.append('80')

    while (len(ascii_list)*8+64)%512 != 0:
        ascii_list.append('00')
        i=i+1
    #add Descriptor

    ascii_list.append(reverse_hex_8bytes(genMsgLengthDescriptor(length_msg_bits)))

    #print "final_ascii_list:\n","".join(ascii_list)
    return "".join(ascii_list)

def getM16(hex_str,operatingBlockNum):

    '''

    --args:

	    hex_str : a hex-encoded string with length in integral multiple of 512bits

	    operatingBlockNum : message block number which is being operated , greater than 1

    --return:

	    M : result of splited 64bytes into 4*16 message blocks with little-endian



    '''

    M = [int(reverse_hex_4bytes(hex_str[i:(i+8)]),16) for i in xrange(128*(operatingBlockNum-1),128*operatingBlockNum,8)]

    return M



#定义函数，用来产生常数T[i]，常数有可能超过32位，同样需要&0xffffffff操作。注意返回的是十进制的数

def T(i):

    result = (int(4294967296*abs(__import__("math").sin(i))))&0xffffffff

    return result   



#定义每轮中用到的函数

#RL为循环左移，注意左移之后可能会超过32位，所以要和0xffffffff做与运算，确保结果为32位

F = lambda x,y,z:((x&y)|((~x)&z))

G = lambda x,y,z:((x&z)|(y&(~z)))

H = lambda x,y,z:(x^y^z)

I = lambda x,y,z:(y^(x|(~z)))

#RL = L = lambda x,n:(((x&lt;&lt;n)|(x&gt;&gt;(32-n)))&(0xffffffff))
RL = L = lambda x,n:(((x<<n)|(x>>(32-n)))&(0xffffffff))



def FF(a, b, c, d, x, s, ac):  

    a = (a+F ((b), (c), (d)) + (x) + (ac)&0xffffffff)&0xffffffff;  

    a = RL ((a), (s))&0xffffffff;  

    a = (a+b)&0xffffffff  

    return a  

def GG(a, b, c, d, x, s, ac):  

    a = (a+G ((b), (c), (d)) + (x) + (ac)&0xffffffff)&0xffffffff;  

    a = RL ((a), (s))&0xffffffff;  

    a = (a+b)&0xffffffff  

    return a  

def HH(a, b, c, d, x, s, ac):  

    a = (a+H ((b), (c), (d)) + (x) + (ac)&0xffffffff)&0xffffffff;  

    a = RL ((a), (s))&0xffffffff;  

    a = (a+b)&0xffffffff  

    return a  

def II(a, b, c, d, x, s, ac):  

    a = (a+I ((b), (c), (d)) + (x) + (ac)&0xffffffff)&0xffffffff;  

    a = RL ((a), (s))&0xffffffff;  

    a = (a+b)&0xffffffff  

    return a      



def show_md5(A,B,C,D):

    return "".join( [  "".join(__import__("re").findall(r"..","%08x"%i)[::-1]) for i in (A,B,C,D)  ]  )

def run_md5(A=0x67452301,B=0xefcdab89,C=0x98badcfe,D=0x10325476,readyMsg=""):
    a = A
    b = B
    c = C
    d = D

    for i in xrange(0,len(readyMsg)/128):

	M = getM16(readyMsg,i+1)

	for i in xrange(16):

	    exec "M"+str(i)+"=M["+str(i)+"]"

	#First round

	a=FF(a,b,c,d,M0,7,0xd76aa478L)

	d=FF(d,a,b,c,M1,12,0xe8c7b756L)

	c=FF(c,d,a,b,M2,17,0x242070dbL)

	b=FF(b,c,d,a,M3,22,0xc1bdceeeL)

	a=FF(a,b,c,d,M4,7,0xf57c0fafL)

	d=FF(d,a,b,c,M5,12,0x4787c62aL)

	c=FF(c,d,a,b,M6,17,0xa8304613L)

	b=FF(b,c,d,a,M7,22,0xfd469501L)

	a=FF(a,b,c,d,M8,7,0x698098d8L)

	d=FF(d,a,b,c,M9,12,0x8b44f7afL)

	c=FF(c,d,a,b,M10,17,0xffff5bb1L)

	b=FF(b,c,d,a,M11,22,0x895cd7beL)

	a=FF(a,b,c,d,M12,7,0x6b901122L)

	d=FF(d,a,b,c,M13,12,0xfd987193L)

	c=FF(c,d,a,b,M14,17,0xa679438eL)

	b=FF(b,c,d,a,M15,22,0x49b40821L)

	#Second round

	a=GG(a,b,c,d,M1,5,0xf61e2562L)

	d=GG(d,a,b,c,M6,9,0xc040b340L)

	c=GG(c,d,a,b,M11,14,0x265e5a51L)

	b=GG(b,c,d,a,M0,20,0xe9b6c7aaL)

	a=GG(a,b,c,d,M5,5,0xd62f105dL)

	d=GG(d,a,b,c,M10,9,0x02441453L)

	c=GG(c,d,a,b,M15,14,0xd8a1e681L)

	b=GG(b,c,d,a,M4,20,0xe7d3fbc8L)

	a=GG(a,b,c,d,M9,5,0x21e1cde6L)

	d=GG(d,a,b,c,M14,9,0xc33707d6L)

	c=GG(c,d,a,b,M3,14,0xf4d50d87L)

	b=GG(b,c,d,a,M8,20,0x455a14edL)

	a=GG(a,b,c,d,M13,5,0xa9e3e905L)

	d=GG(d,a,b,c,M2,9,0xfcefa3f8L)

	c=GG(c,d,a,b,M7,14,0x676f02d9L)

	b=GG(b,c,d,a,M12,20,0x8d2a4c8aL)

	#Third round

	a=HH(a,b,c,d,M5,4,0xfffa3942L)

	d=HH(d,a,b,c,M8,11,0x8771f681L)

	c=HH(c,d,a,b,M11,16,0x6d9d6122L)

	b=HH(b,c,d,a,M14,23,0xfde5380c)

	a=HH(a,b,c,d,M1,4,0xa4beea44L)

	d=HH(d,a,b,c,M4,11,0x4bdecfa9L)

	c=HH(c,d,a,b,M7,16,0xf6bb4b60L)

	b=HH(b,c,d,a,M10,23,0xbebfbc70L)

	a=HH(a,b,c,d,M13,4,0x289b7ec6L)

	d=HH(d,a,b,c,M0,11,0xeaa127faL)

	c=HH(c,d,a,b,M3,16,0xd4ef3085L)

	b=HH(b,c,d,a,M6,23,0x04881d05L)

	a=HH(a,b,c,d,M9,4,0xd9d4d039L)

	d=HH(d,a,b,c,M12,11,0xe6db99e5L)

	c=HH(c,d,a,b,M15,16,0x1fa27cf8L)

	b=HH(b,c,d,a,M2,23,0xc4ac5665L)

	#Fourth round

	a=II(a,b,c,d,M0,6,0xf4292244L)

	d=II(d,a,b,c,M7,10,0x432aff97L)

	c=II(c,d,a,b,M14,15,0xab9423a7L)

	b=II(b,c,d,a,M5,21,0xfc93a039L)

	a=II(a,b,c,d,M12,6,0x655b59c3L)

	d=II(d,a,b,c,M3,10,0x8f0ccc92L)

	c=II(c,d,a,b,M10,15,0xffeff47dL)

	b=II(b,c,d,a,M1,21,0x85845dd1L)

	a=II(a,b,c,d,M8,6,0x6fa87e4fL)

	d=II(d,a,b,c,M15,10,0xfe2ce6e0L)

	c=II(c,d,a,b,M6,15,0xa3014314L)

	b=II(b,c,d,a,M13,21,0x4e0811a1L)

	a=II(a,b,c,d,M4,6,0xf7537e82L)

	d=II(d,a,b,c,M11,10,0xbd3af235L)

	c=II(c,d,a,b,M2,15,0x2ad7d2bbL)

	b=II(b,c,d,a,M9,21,0xeb86d391L)



	A += a

	B += b

	C += c

	D += d



	A = A&0xffffffff

	B = B&0xffffffff

	C = C&0xffffffff

	D = D&0xffffffff


	a = A

	b = B

	c = C

	d = D

    return show_md5(a,b,c,d)
```

## 参考

[phpwind 利用哈希长度扩展攻击进行getshell](https://www.leavesongs.com/PENETRATION/phpwind-hash-length-extension-attack.html)
[Understanding MD5 Length Extension Attack](http://blog.chinaunix.net/uid-27070210-id-3255947.html)
[Length_extension_attack](https://en.wikipedia.org/wiki/Length_extension_attack)
[论野生技术&二次元](https://yooooo.us/2013/python-endian-conversion?variant=zh-cn)

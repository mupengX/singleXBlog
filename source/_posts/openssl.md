title: OpenSSL
date: 2016-06-19 01:27:30
description: OpenSSL
categories: 
tags: opnessl

---

# SSL
----------
SSL 是一个缩写，代表的是 Secure Sockets Layer。它是支持在 Internet 上进行安全通信的标准，并且将数据密码术集成到了协议之中。数据在离开您的计算机之前就已经被加密，然后只有到达它预定的目标后才被解密。证书和密码学算法支持了这一切的运转，如果连接传输敏感信息，则应使用 SSL。

# OpenSSL
----------
openssl是一个开源程序的套件、这个套件有三个部分组成：

- 一是libcryto，这是一个具有通用功能的加密库，里面实现了众多的加密库；
- 二是libssl，这个是实现ssl机制的，它是用于实现TLS/SSL的功能；
- 三是openssl，是个多功能命令行工具，它可以实现加密解密，甚至还可以当CA来用，可以让你创建证书、吊销证书。

要获得关于如何使用 OpenSSL 命令行工具的资料，请参阅[官方手册][1]


  [1]: https://www.openssl.org/docs/manmaster/apps/openssl.html
  
# 密钥、证书请求、证书概要说明
----------

在申请证书过程中，涉及到密钥，证书请求，证书这几个概念，它们之间的联系是：

-  首先，生成客户端的密钥，即客户端的公私钥对，且要保证私钥只有客户端自己拥有。
-  然后以客户端的密钥和客户端自身的信息(国家、机构、域名、邮箱等)为输入，生成证书请求文件。其中客户端的公钥和客户端信息是明文保存在证书请求文件中的，而客户端私钥的作用是对客户端公钥及客户端信息做签名，自身是不包含在证书请求中的。然后把证书请求文件发送给CA机构。
-  最后CA机构接收到客户端的证书请求文件后，首先校验其签名，然后审核客户端的信息，最后CA机构使用自己的私钥为证书请求文件签名，生成证书文件，下发给客户端。此证书就是客户端的身份证，来表明用户的身份。

其中，证书签发机构CA，CA是被绝对信任的机构。
自签名证书就是自己给自己签发的证书，例如12306网站，它用的自签名证书，以为12306不是浏览器所信任的证书签发机构，所以浏览器会有提示证书有问题，将其的证书导入到电脑中，浏览器就不会报该错误。除非特别相信某个机构，否则不要在机器上随便导入证书，这种做法存在安全风险。


# OpenSSL使用
---------

生成RSA密钥对。使用DES3加密，密钥使用密码保护，长度为1024，输出到rsaprivatekey.pem

``` 
openssl genrsa -des3 -out rsaprivatekey.pem -passout pass:123456 1024

```
openssl可以将这个文件中的公钥提取出来:

```
openssl rsa -in rsaprivatekey.pem -pubout -out test_pub.key

```

当使用-new选取的时候，说明是要生成证书请求，当使用x509选项的时候，说明是要生成自签名证书。

生成证书请求文件

```
openssl req -new -key rsaprivatekey.pem -out rsaCertReq.csr

```
或者生成1024位RSA密钥，并生成证书请求文件

```
openssl req -new -newkey rsa:1024 -out rsaCertReq.csr -keyout RSA.pem -batch
```

生成自签证书并指定过期时间

```
openssl x509 -req -days 3650 -in rsaCertReq.csr -signkey rsaprivatekey.pem -out rsaCert.crt
```
或者

```
openssl req -new -x509 -key rsaprivatekey.pem -out rsaprivatekey.cert -days 1095
```
证书里有对应的公钥和国家、地区等信息。验证证书：

```
openssl verify rsaprivatekey.cert
```

从证书中提取公钥：

```
openssl x509 -in rsaprivatekey.cert  -noout -pubkey > pubkey.pem
```

生成pem结尾的私钥供Java使用:

```
openssl pkcs8 -topk8 -inform PEM -outform DER -in ca.key.pem -out ca.private.der -nocrypt
```

# 举个栗子
---------

产生1024位RSA私匙，用3DES加密它，口令为123456。输出到文件rsaprivatekey_pass.pem：

```
openssl genrsa -out rsaprivatekey_pass.pem -passout pass:123456 -des3 1024
```

生成证书,days为有效天数，输出证书文件到rsaprivatekey.cert：

```
openssl req -new -x509 -key rsaprivatekey_pass.pem -out rsaprivatekey.cert -days 1095 -passin pass:123456
```
此证书文件中包含公钥，用于和对方进行公开的交换。

使用私钥对test.txt文本内容进行数字签名，输出到test.sign

```
openssl rsautl -sign -in test.txt -out test.sign -inkey rsaprivatekey_pass.pem -passin pass:123456
```
使用公钥证书对数字签名进行验证，输出到test.vfy，此时test.vfy和test.txt的内容应完全一样:

```
openssl rsautl -verify -in test.sig -out test.vfy -inkey rsaprivatekey.cert -certin

```

# 最后
------
这里只是简单介绍了openssl的一些用法，其他内容推荐查看官方文档。

关于数字签名的详细介绍，推荐[阮一峰的一篇日志][2]
[2]:http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html
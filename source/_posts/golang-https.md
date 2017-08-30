title: go语言HttpClient中的https证书
date: 2017-08-18 17:17:15

categories: Go

tags: [Go, https]

---

关于HTTPS中SSL/TLS具体原理就不具体说了，可以看一下阮一峰老师的[文章](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)

这里我们主要说一下HTTPS中的证书，HTTPS开始的握手阶段是基于非对称加密，也就是公钥加密的数据要用私钥才能解开，同理用私钥加密的数据用公钥才能解开。

然而有一肚子坏水的家伙可以进行中间人劫持然后用自己的公钥来替换要访问的服务器的公钥，这样本来要发给服务器的数据发给了中间人，信息就被他们给窃取走了，所以有了CA这个机构来负责管理和签发证书。CA会将申请者的一些信息和公钥用CA自己的私钥加密并附上CA的一些信息生成数字证书给申请者。client在向服务器发出加密请求的时候，服务器会将数字证书一起返回给client，client在受信任的根证书颁发机构列表里来查看解开数字证书的公钥是否在列表之内，如果这张数字证书不是由受信任的机构颁发的会发出警告，如果数字证书是可靠的，客户端就可以使用证书中的服务器公钥。

通过CA的数字签名来保证公钥的有效性，而黑客的公钥是很难通过CA的认证。CA证书具有层级结构，它建立了自上而下的信任链，下级CA信任上级CA，下级CA由上级CA颁发证书并认证。数字证书最常见的格式是X.509，这种公钥基础设施又称之为PKIX。在用HttpClient进行数字证书认证出错的时候会提示：`PKIX path building failed`

我们知道浏览器保存了一个常用的CA证书列表，那么用golang的HttpClient请求HTTPS的服务时，它所受信任的证书列表在哪里呢？

查看golang的源码发现在目录`src/crypto/x509`下有针对各个操作系统获取根证书的实现，例如root_linux.go中记录了各个Linux发行版根证书的存放路径：

```golang
  // Copyright 2015 The Go Authors. All rights reserved.
  // Use of this source code is governed by a BSD-style
  // license that can be found in the LICENSE file.
  
  package x509
  
  // Possible certificate files; stop after finding one.
  var certFiles = []string{
  	"/etc/ssl/certs/ca-certificates.crt",                // Debian/Ubuntu/Gentoo etc.
  	"/etc/pki/tls/certs/ca-bundle.crt",                  // Fedora/RHEL 6
  	"/etc/ssl/ca-bundle.pem",                            // OpenSUSE
  	"/etc/pki/tls/cacert.pem",                           // OpenELEC
  	"/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem", // CentOS/RHEL 7
  }
  
```

root_unix.go中写了类Unix系统的证书路径：

```golang
// Possible directories with certificate files; stop after successfully
// reading at least one file from a directory.
  var certDirectories = []string{
  	"/etc/ssl/certs",               // SLES10/SLES11, https://golang.org/issue/12139
  	"/system/etc/security/cacerts", // Android
  }
  

```

对应的还有root_darwin.go，root_windows.go等分别实现了MacOS及Windows平台下的根证书的获取。

另外，Java在JRE的安装目录下也有一份默认的可信任证书列表，这个列表一般是保存在 $JRE/lib/security/cacerts 文件中，随着JDK版本的升级而更新。
title: Go与TLS的那些事

toc: true

date: 2017-10-21 17:22:27

categories: Go

tags: [TLS,Go]

---

安全一直是一件很重要的事情，现在大部分正经公司的网站都已经跑在HTTPS上。所以这里分享一下如何用Go创建自签名证书并用自签名证书来实现一个支持HTTPS的服务。

### 公私钥加解密
公私钥加密除了能保证内容的安全性以外还用来证明你自己是你自己，因为只有用你的私钥才能解密由你公钥加密的内容，公钥是对所有人公开，而私钥只有你自己知道。

在Go中有一个`crypto/rsa`包，提供了非对称加密的实现，首先我们生成一对公私钥：

```go
// NOTE: Use crypto/rand not math/rand
privKey, err := rsa.GenerateKey(rand.Reader, 2048)
if err != nil {
    log.Fatalf("generating random key: %v", err)
}
```
用公钥加密的内容只能用对应的私钥解开，主要原理是基于大质数的分解，有兴趣的可以查看相关资料，这里先按下不表。

用公钥加密：

```go
plainText := []byte("Hi I'm Xiaoming")
// use the public key to encrypt the message
cipherText, err := rsa.EncryptPKCS1v15(rand.Reader, &privKey.PublicKey, plainText)
if err != nil {
    log.Fatalf("could not encrypt data: %v", err)
}
```

这样我们就得到了密文`cipherText`，如果你打印密文的话将看到一串类似于乱码的内容。

用秘钥解密：

```go
decryptedText, err := rsa.DecryptPKCS1v15(nil, privKey, cipherText)
if err != nil {
	log.Fatalf("error decrypting cipher text: %v", err)
}
fmt.Printf("%s\n", decryptedText)

```
那么如何证明你自己是你自己呢？如果我想要跟你通信首先要证明你的身份，我用你的公钥来加密一个内容发送给你，然后你用私钥解开后将内容返回给我，如果你能将我发送的内容正确无误的返回给我那就证明即将要跟我通信的人确实是你。

但是，如果我获取了公钥的时候被中间人攻击，我拿到的公钥不是你的而是中间人的怎么办呢？这样我会误认为中间人是你。这个时候就需要CA证书了。

### 数字签名

数字签名的作用是用来校验内容的正确性，保证内容没有被篡改过。

数字签名生成的过程是对要发送的内容来做hash得到一个指纹，然后用私钥对指纹进行计算得到签名。

验证签名的过程是，对接到内容做相同的hash得到一个指纹，用私钥对应的公钥来得到签名里的指纹信息，比对两个指纹，如果两个指纹能对上则证明内容没有被篡改过。

有人会问为什么不直接对发送的内容算签名而是要先得到一个hash值，首先hash计算得到的值比较短，对这个值进行签名计算比较快。另外RSA在计算签名时对签名内容的长度也有限制，如果直接将一本《红楼梦》的内容来计算签名那是很难想象的。

同样我们可以用`crypto/rsa`包来计算签名：

```go
plainText:="Hi, I'm Xiaoming"
h := sha1.New()
h.Write([]byte(plainText))
digest := h.Sum(nil)
fmt.Printf("The hash of my message is: %s\n", string(digest))

// generate a signature using the private key
signature, err := rsa.SignPKCS1v15(rand.Reader, privKey, crypto.SHA1, digest)
if err != nil {
    log.Fatalf("error creating signature: %v", err)
}

```

验证签名：

```go
func Verify(pub *rsa.PublicKey, data, signature []byte) error {
	h := sha1.New()
	h.Write(data)
	digest := h.Sum(nil)
	return rsa.VerifyPKCS1v15(pub, crypto.SHA1, digest, signature)
}
```
那么对于之前说的中间人替换了证书的情况，就可以用数字签名来解决。首先服务提供方需要去wellknown CA那里申请一张证书，这个证书的内容是服务提供方的相关信息和公钥，然后用CA的私钥对这些内容做一个签名附在证书里。在建立HTTPS连接的时候服务端需要向客户端提供他的证书，客户端首先验证证书的合法性：由wellknow CA签发，并且通过数字签名校验证书内容的正确性，没有问题后就可以认为证书的公钥确实是服务端的公钥而不是被中间人篡改的其他公钥。

### 自签名证书&HTTPS服务

在开发的过程中我们可能需要先用一个自签名的证书来满足开发需求，或者我们要自建CA来为其他服务签发证书(k8s中可以用自签证书来做认证)这个时候就可以用`crypto/x509`来生成一张自签名的证书。

首先要创建生成证书的请求：

```go
func CertTemplate() (*x509.Certificate, error) {
    serialNumberLimit := new(big.Int).Lsh(big.NewInt(1), 128)
    serialNumber, err := rand.Int(rand.Reader, serialNumberLimit)
    if err != nil {
        return nil, errors.New("failed to generate serial number: " + err.Error())
    }

    tmpl := x509.Certificate{
        SerialNumber:          serialNumber,
        Subject:               pkix.Name{Organization: []string{"Siglecool, Inc."}},
        SignatureAlgorithm:    x509.SHA256WithRSA,
        NotBefore:             time.Now(),
        NotAfter:              time.Now().Add(time.Hour),         BasicConstraintsValid: true,
    }
    return &tmpl, nil
}
```
接下来需要创建一对公私钥，并完善生成证书请求的内容：

```go
rootKey, err := rsa.GenerateKey(rand.Reader, 2048)
if err != nil {
	log.Fatalf("generating random key: %v", err)
}

rootCertTmpl, err := CertTemplate()
if err != nil {
	log.Fatalf("creating cert template: %v", err)
}
// describe what the certificate will be used for
rootCertTmpl.IsCA = true
rootCertTmpl.KeyUsage = x509.KeyUsageCertSign | x509.KeyUsageDigitalSignature
rootCertTmpl.ExtKeyUsage = []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth, x509.ExtKeyUsageClientAuth}
rootCertTmpl.IPAddresses = []net.IP{net.ParseIP("127.0.0.1")}

```
接下来创建根证书：

```go
func CreateCert(template, parent *x509.Certificate, pub interface{}, parentPriv interface{}) (
    cert *x509.Certificate, certPEM []byte, err error) {

    certDER, err := x509.CreateCertificate(rand.Reader, template, parent, pub, parentPriv)
    if err != nil {
        return
    }
    
    cert, err = x509.ParseCertificate(certDER)
    if err != nil {
        return
    }
    b := pem.Block{Type: "CERTIFICATE", Bytes: certDER}
    certPEM = pem.EncodeToMemory(&b)
    return
}

rootCert, rootCertPEM, err := CreateCert(rootCertTmpl, rootCertTmpl, &rootKey.PublicKey, rootKey)
```
然后可以用根证书来签发httpserver的证书：

```go
servKey, err := rsa.GenerateKey(rand.Reader, 2048)
if err != nil {
	log.Fatalf("generating random key: %v", err)
}

// create a template for the server
servCertTmpl, err := CertTemplate()
if err != nil {
	log.Fatalf("creating cert template: %v", err)
}
servCertTmpl.KeyUsage = x509.KeyUsageDigitalSignature
servCertTmpl.ExtKeyUsage = []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth}
servCertTmpl.IPAddresses = []net.IP{net.ParseIP("127.0.0.1")}

_, servCertPEM, err := CreateCert(servCertTmpl, rootCert, &servKey.PublicKey, rootKey)
```
用httpserver证书来创建HTTPS服务：

```go
servKeyPEM := pem.EncodeToMemory(&pem.Block{
	Type: "RSA PRIVATE KEY", Bytes: x509.MarshalPKCS1PrivateKey(servKey),
})
servTLSCert, err := tls.X509KeyPair(servCertPEM, servKeyPEM)
if err != nil {
	log.Fatalf("invalid key pair: %v", err)
}
handler := func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("Hello World!")) }
s := httptest.NewUnstartedServer(http.HandlerFunc(handler))
s.TLS = &tls.Config{
	Certificates: []tls.Certificate{servTLSCert},
}

```
这个时候请求这个server会报错：

```go
s.StartTLS()
_, err = http.Get(s.URL)
s.Close()
fmt.Println(err)
// x509: certificate signed by unknown authority
```
由于我们的证书不是知名CA签发的所以在client请求serve时在校验证书环节会报错，同时服务端也会提示：`http: TLS handshake error from 127.0.0.1:53844: remote error: bad certificate`

为了让client信任server的证书，我们需要在client中用我们的root证书来替代系统的root证书，因为server的证书使用我们自己生成的root证书签发的，这样server证书就可以验证通过：

```go
certPool := x509.NewCertPool()
certPool.AppendCertsFromPEM(rootCertPEM)
client := &http.Client{
	Transport: &http.Transport{
		TLSClientConfig: &tls.Config{RootCAs: certPool},
	},
}

s.StartTLS()
resp, err := client.Get(s.URL)
s.Close()
if err != nil {
	log.Fatalf("could not make GET request: %v", err)
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
if err != nil {
	log.Fatalf("could not response: %v", err)
}
fmt.Println(string(body))
```
这次我们就可以收到`Hello World`啦。
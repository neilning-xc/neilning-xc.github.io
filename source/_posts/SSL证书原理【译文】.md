---
title: SSL证书原理【译文】
author: Neil Ning
date: 2024-06-23 17:33:08
tags: ['https', 'ssl', 'certificate']
categories: 翻译
cover: bg.jpg
---
## 前言
该文章翻译自【Dissecting an SSL certificate】，[点击这里](https://jvns.ca/blog/2017/01/31/whats-tls/)查看原文。以下是原文内容。

Hello，在我个人的网络杂志上有一个HTTPS的页面，当我打算只用200个字来解释这个事情的时候，我发现有更多的事情值得探讨。所以在这篇文章中我们将会剖析一下SSL证书，来试图真正的理解他的原理。
我不是一个网络安全人员，所以我不会为你的网站提供网络安全建议。但是我认为了解SSL证书相关的问题还是很有趣的，我可以讨论一下这个。

## TLS：新版的SSL
曾经在很久的一段时间内，我对TSL感到很困惑。基本上新版的SSL称之为TSL（SSL 3.0之后的版本称为TLS 1.0）。为了减少困惑，在本文中我打算一直称它为SSL。
## 什么是证书
假设我正在访问https://mail.google.com查看我的邮箱。`mail.google.com`是一个运行在HTTPS服务器的443端口上的应用。我想确保我访问的是真正的mail.google.com，而不是由其他人恶意仿造的网站。
在很长一段时间里证书这个词曾让我感到很困惑，直到我的通知Ray告诉我，我可以通过命令行来链接一个服务器并下载他的证书。
如果你只是想看一下SSL证书，可以在浏览器里点击绿色的锁，你可以看到所有关于证书的信息。但是通过命令行的方式也很有趣。
所以让我们来看一下mail.google.com的证书，并解析一下它。
首先，运行`openssl s_client -connect mail.google.com:443`
该命令会打印很多东西，但是我们仅关注关于证书的内容，输出如下：
```
$ openssl s_client -connect mail.google.com:443
...
-----BEGIN CERTIFICATE-----
MIIElDCCA3ygAwIBAgIIMmzfdZnO9pMwDQYJKoZIhvcNAQELBQAwSTELMAkGA1UE
BhMCVVMxEzARBgNVBAoTCkdvb2dsZSBJbmMxJTAjBgNVBAMTHEdvb2dsZSBJbnRl
cm5ldCBBdXRob3JpdHkgRzIwHhcNMTcwMTE4MTg1MjExWhcNMTcwNDEyMTg1MDAw
WjBpMQswCQYDVQQGEwJVUzETMBEGA1UECAwKQ2FsaWZvcm5pYTEWMBQGA1UEBwwN
TW91bnRhaW4gVmlldzETMBEGA1UECgwKR29vZ2xlIEluYzEYMBYGA1UEAwwPbWFp
bC5nb29nbGUuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAiYcr
C9Rn7g9xjsg7khqfRPxUnvpgGyCHqJMXxZGtdf+G02d07cPlMEeaGG12vHyVfRZD
tc/F1ZfwenH6gf0uMobtgw7n2NQa7T7qxuqSUDhZsO1sI1LL/Yqy8QHoooOZQWMz
ytuRA18zti4vQV1dCijADh0+NWI1GDUAKidbaH/fBRrStqBev5Bhq3ZaGj3fDjAO
7CG0Wk3n4Ov2yg44XOdgkLMzjdnbV8l6cZDC7lCK1VsEU1mEd0O0Dw4OcnHLuBPw
IkioZayhPOXDXUS+bhpmtEiCkt8kbHG6jNMC4m8t62Jaf/Si3XNcHhDa4wPCTvid
X//PuuNlRZVg3NjK/wIDAQABo4IBXjCCAVowHQYDVR0lBBYwFAYIKwYBBQUHAwEG
CCsGAQUFBwMCMCwGA1UdEQQlMCOCD21haWwuZ29vZ2xlLmNvbYIQaW5ib3guZ29v
Z2xlLmNvbTBoBggrBgEFBQcBAQRcMFowKwYIKwYBBQUHMAKGH2h0dHA6Ly9wa2ku
Z29vZ2xlLmNvbS9HSUFHMi5jcnQwKwYIKwYBBQUHMAGGH2h0dHA6Ly9jbGllbnRz
MS5nb29nbGUuY29tL29jc3AwHQYDVR0OBBYEFI69aYCEtb2swbJJR3cMOTdcfvZ4
MAwGA1UdEwEB/wQCMAAwHwYDVR0jBBgwFoAUSt0GFhu89mi1dvWBtrtiGrpagS8w
IQYDVR0gBBowGDAMBgorBgEEAdZ5AgUBMAgGBmeBDAECAjAwBgNVHR8EKTAnMCWg
I6Ahhh9odHRwOi8vcGtpLmdvb2dsZS5jb20vR0lBRzIuY3JsMA0GCSqGSIb3DQEB
CwUAA4IBAQAhiqQIwkGp1NmlLq89gjoAfpwaapHuRixxl2S54fyu/4WOHJJafqVA
Tya9J7GTUCyQ6nszCdVizVP26h9TKOs9LJw5jWV9SOnPU2UZKvrNnOUi2FUkCcuD
lsADdKSXNzye3jB88TENrWC/y3ysPdAgPO/sXzyRvNw8SVKl2+RqMDpSRpBptF9e
Lp+WLAM3xKS5SPwCNdCiA332o7qiKRKQm/6bbIWnm7hp/ZnLxbyKaIVytRdiwRNp
O/TTpRv2C708GA3PH6i1pYE86xm3w7lGhN9OiCZpKOJD6ZUH3W20idgPKYPBCO/N
Op2AF3I4iUGeQjXFVLgS6mjUvdLndL9G
-----END CERTIFICATE-----
```
目前输出的内容还不是一段有意义的内容，它的格式被称为**X509**。openssl命令可以解析他。
接下来我把它保存到一个文本文件中，并将它命名为`cert.pem`。如果你打算跟着一步一步尝试，你可以从[这里](https://gist.githubusercontent.com/jvns/2c249b8059c0b183c02bb3a71e12e418/raw/b4afd1877471b72f8900b2396c30bc670a5701c9/mail_google_cert.pem)将它保存到自己的电脑中。
下一步，我们来解析这个证书文件。
```
$ openssl x509 -in cert.pem -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3633524695565792915 (0x326cdf7599cef693)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Google Inc, CN=Google Internet Authority G2
        Validity
            Not Before: Jan 18 18:52:11 2017 GMT
            Not After : Apr 12 18:50:00 2017 GMT
        Subject: C=US, ST=California, L=Mountain View, O=Google Inc, CN=mail.google.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:89:87:2b:0b:d4:67:ee:0f:71:8e:c8:3b:92:1a:
                    9f:44:fc:54:9e:fa:60:1b:20:87:a8:93:17:c5:91:
                    .... blah blah blah ............
                    c2:4e:f8:9d:5f:ff:cf:ba:e3:65:45:95:60:dc:d8:
                    ca:ff
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:mail.google.com, DNS:inbox.google.com

            X509v3 Subject Key Identifier: 
                8E:BD:69:80:84:B5:BD:AC:C1:B2:49:47:77:0C:39:37:5C:7E:F6:78
    Signature Algorithm: sha256WithRSAEncryption
         21:8a:a4:08:c2:41:a9:d4:d9:a5:2e:af:3d:82:3a:00:7e:9c:
         1a:6a:91:ee:46:2c:71:97:64:b9:e1:fc:ae:ff:85:8e:1c:92:
         ......... blah blah blah more goes here ...........
```
文件中包含了很多内容，下面是各个部分的解释。
- `CN=mail.google.com`表示“common name”，与直觉相反的是，你可以忽略这个字段，并关注“subject alternative name”字段。
- expiry date指的是过期时间
- `X509v3 Subject Alternative Name`字段的内容是该证书的域名列表，这表示该证书在这这两个域名下都可以正常工作。
- `Public Key Info`包含了我们后续和mail.google.com通信的公钥，我们不会花时间解释公钥的密码学含义，基本上它是一个加密密钥，保证通信的安全。
- `signature`签名算法，它是一个很重要的东西，基本上任何人都可以为mail.google.com这个域名制作一个证书，但是我现在签发一个证书，你肯定不会信任他。

接下来我们来讨论一下证书签名。

## 证书签名
每个互联网证书都会由两部分组成
1. 证书本身（包括要验证的域名、公钥和其他信息）
2. 由其他机构签署的签名，就好比经过银行认证的信用卡

在我计算机的`/etc/ssl/certs`目录下有很多证书，这些证书都是我本机信任的证书，他们可以用来签署其他证书。比如，我的笔记本上有个证书：`/etc/ssl/certs/Staat_der_Nederlanden_EV_Root_CA.pem`。这是一个来自荷兰的根证书，假如用它为`mail.google.com`域名签署得到一个证书，我的电脑就会信任这个域名下的证书。

但如果是由其他人任意签署的证书，我的电脑则不会信任并拒绝这个证书。

`mail.google`的证书由下面的证书签署
1. s:/C=US/ST=California/L=Mountain View/O=Google Inc/CN=mail.google.com
2. 这个证书由`Google Internet Authority G2`证书签署认证
3. 第二步的证书由`GeoTrust Global CA`签署认证
4. 第三步的证书由`Equifax Secure Certificate Authority`签署认证

我的电脑上有`/etc/ssl/certs/GeoTrust_Global_CA.pem`文件，所以电脑会信任`mail.google.com`的证书。

## 证书是如何签发的
签发的证书的过程大致如下：
1. 生成证书的前半部分（包括域名、过期时间、公钥等），这部分是公开的。
2. 同时，为证书生成一个私钥，私钥必须保证安全，不能展示给任何人。每次建立SSL连接的时候会用到这个私钥。
3. 证书权威机构为你签署证书，证书权威机构应该受信任的机构，所有当它们为某个域名签署证书的时候，必须确认提交者真正拥有这个域名。
4. 用签署的证书配置你的网站，用它来证明用户访问网站的真实性。

从上面的过程可以看到“证书权威机构应该是值得信任的”是整个体系的关键，所以当有的证书签发机构不负责任的为没有拥有某个域名的人签发证书时，人们会感到愤怒。

## 证书透明度

我们讨论的最后主题是[证书透明度（Certificate Transparency）](https://certificate.transparency.dev/)，这是一个有趣的主题，并且有一个官方网站来解释，我会尽量解释这个主题。
上面我们说过证书权威机构应该是值得信任的，我们的计算机信任了很多权威证书，他们中任意一个都可能会为`mail.google.com`签署流氓证书，这是不安全的。
这并不是一个假设，[证书透明度网站](https://certificate.transparency.dev/)就有不止一个案例，某些权威的证书签发机构犯的错误。
整个过程大概是这样的，对于`mail.google.com`，Google知道所有的经过验证的证书（可能有一个或者多个），证书透明度会确保如果存在一个没有被记录的`mail.google.com`的证书，他们能够发现这是一个非法证书。
大致步骤如下：
1. CA机构每次签署一个证书，他们需要将这个证书放入一个全球公开的**证书日志**中
2. 不仅如此，谷歌机器人也会将它在网络上发现的所有证书放入**证书日志**中
3. 如果一个证书不在证书日志中，浏览器将不会接受这个证书
4. 任何人可以在任何时候查询一个证书是否在证书日志中

所以如果荷兰CA机构签发了恶意的mail.google.com证书，他们还需要将这个证书放到证书日志中，但是谷歌能够发现这个恶意的行为。证书不会被加到日志中，浏览会拒绝该证书。
## SSL配置并不容易
好了，我们已经下载了一个SSL证书并深入讨论了一些证书的相关知识，希望你学到了一些东西。
为web服务器设置SSL证书和SSL配置是令人困惑的，据我了解SSL有很多配置项。[SSL Labs上有对mail.google.com的检测结果](https://www.ssllabs.com/ssltest/analyze.html?d=mail.google.com&s=216.58.194.165&hideResults=on)，检测结果有包含像`OLD_TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`这样的配置项，我很高兴有项SSL Labs这样的工具来帮助我们凡人理解这些配置。
如果你不确定如何配置SSL，可以参考[这里的例子](https://syslink.pl/cipherlist/)进行配置，不过我不确定这些配置的好坏。
## let’s encrypt
let’s encrypt是个很酷的网站。你甚至不需要理解证书的相关原理，它就可以为你的网站生成安全的证书，重要的是，它还是免费的。


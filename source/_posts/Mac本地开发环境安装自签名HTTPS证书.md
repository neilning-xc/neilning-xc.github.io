---
title: 本地开发环境生成自签名证书，启用HTTPS
date: 2017-12-10 23:11:40
tags: ["Mac", "HTTPS"]
---

## 0. 前言
今天一大早上班Chrome浏览器提示已经自动升级（Version 63）需要重启浏览器，重启之后发现本地的开发环境打开不，原因是新版的浏览器强制将http转换成https了，而我本地的开发环境没有启用https。解决方案有两种：
1. 将本地开发环境的域名（例如test.local）加入浏览的一个白名单，告诉浏览器该域名不需要强制启用https.
2. 在本地生成一个自签名证书，并启用https.

第一种方式比较简单，在浏览器地址栏中输入chrome://net-internals/#hsts。在Delete domain 栏的输入框中输入要http访问的域名，然后点击delete地按钮，即可完成配置。然后你可以在Query domain栏中搜索刚才输入的域名，点击逗query地按钮后如果提示逗Not found即为成功。但是，我同事们用此方法都成功了，就我没有成功，无奈。。。遂研究第二种解决方案。

对于第二种解决方案，网上找了几个中文教程，但是新版本的浏览器已经不再适用。最后还是在强大的Google的帮助下找到此篇英文博客.[https://deliciousbrains.com/https-locally-without-browser-privacy-errors/](https://deliciousbrains.com/https-locally-without-browser-privacy-errors/) 有兴趣的可以阅读原文，本文也并没有逐字翻译原文。教程也只适用于Mac系统，Windows系统还没有研究。

本教程假设本地域名为`test.local`

## 1. 生成自签名证书
- 创建生成证书所需的配置文件

创建生成证书所需的配置文件，文件内容如下：

```
[ req ]

default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]

countryName                 = Country Name (2 letter code)
countryName_default         = US

stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = NY

localityName                = Locality Name (eg, city)
localityName_default        = New York

organizationName            = Organization Name (eg, company)
organizationName_default    = Example, LLC

commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_default          = Example Company

emailAddress                = Email Address
emailAddress_default        = test@example.com

[ x509_ext ]

subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]

subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]

DNS.1       = test.local
```
注意将文件的最后一行的改成自己的域名`DNS.1=test.local`。将文件保存并命名为`test.local.conf`。

- 生成证书，在命令行中运行：
```
openssl req -config test.local.conf -new -sha256 -newkey rsa:2048 -nodes -keyout test.local.key -x509 -days 365 -out test.local.crt
```
生成证书时，会有一系列问题需要填写，其他的问题都可以敲回车直接跳过，只将common name填写成你的域名，例如：
```
Common Name (e.g. server FQDN or YOUR name) []:test.local
```
命令运行成功会在当前目录下生成两个文件：`test.local.crt`, `test.local.key`

## 2. Nginx配置
关键的Nginx配置如下，其他部分省略：
```
server {
    listen 80;
    listen 443 ssl http2;
    server_name test.local;
    
    ssl on
    ssl_certificate     /etc/nginx/ssl/test.local.crt;
    ssl_certificate_key /etc/nginx/ssl/test.local.key;
    
    ...
}
```

## 3. 本机配置
重启nginx后，打开Chrome浏览器输入https://test.local，此时浏览器应该会提示Your connection is not private。打开浏览器调试工具，选择security选项卡，显示如下：

![image](image1.png)

将红框中的证书图标拖到桌面，会在桌面生成一个以cer为后缀的文件，双击文件，打开Keychain Access（需要输入密码）
![image](image2.png)

之后会打开一个列表：
![image](image3.png)

找到test.local的证书并双击，打开如下对话框：
![image](image4.png)

点击红框中的下拉菜单，将其设置为Always trust，然后关闭对话框（会再次要求输入密码确认）。完成之后，刷新浏览器页面即可正常打开，并且显示已经正常启用https。

参考文章：
1. https://deliciousbrains.com/https-locally-without-browser-privacy-errors/
2. https://zhidao.baidu.com/question/1435378862579313259.html

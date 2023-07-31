---
title: Web Push协议【译】
author: Neil Ning
date: 2023-07-31 23:15:13
tags: ['web push', 'push notification', 'JWT', 'ECDH', 'HKDF']
cover: bg.jpeg
---
## 前言
该文章翻译自web.dev的[Notifications](https://web.dev/notifications/)系列文章，本篇文章原文链接[点击这里](https://web.dev/push-notifications-web-push-protocol/)

我们已经看到web push库是如何实现消息推送的，但是这些具体帮我们做了什么？
这些库帮我们发送网络请求，并确保请求的格式是符合规范要求的。[Web Push Protocol](https://tools.ietf.org/html/draft-ietf-webpush-protocol)定义了这个网络请求的规范。
![request.jpg](request.jpg)

本章我们来介绍一下服务器如何使用应用服务器密钥来标识自己，如何加密要发送的数据。
这是web push中比较难的一部分，并且我不是加密方面的专家。让我们来探讨其中的每个步骤，理解这些库在背后做了些什么是很有必要的。
## 应用服务器密钥（Application server keys）
当我们订阅用户时，我们传入了一个`applicationServerKey`参数，这个key会被发送给推送服务，他能让推送服务识别出订阅的用户时哪个应用的，以及应用触发的推送应该发送给订阅的用户。
当我们触发消息推送时，我们要设置一些列请求头，这使得推送服务能够对发送的应用进行验证。（[VAPID规范](https://tools.ietf.org/html/draft-thomson-webpush-vapid)定义了这个过程）
在整个过程中到底发生了什么？以下是应用服务器认证的步骤：
1. 应用服务器使用私钥对要发送的JSON数据进行签名。
2. 签名后的数据放在POST请求的请求头中被发送给推送服务。
3. 推送服务使用已经存储的公钥对收到的签名数据进行验证。该公钥是用户在浏览器端订阅时调用`pushManager.subscribe()`时传给推送服务的，且该公钥和第一步签名的私钥时相对应的。
4. 如果验证通过，推送服务把消息发送给用户。

整个过程时示意图如下（注意左下角图例中的公钥和私钥）：
![serverapp-key.jpg](serverapp-key.jpg)

请求头中的签名数据就是JSON Web Token.
### JSON web token
JSON web token简称JWT，我们将数据发送给第三方时，通过JWT对方能够识别出数据的发送者。
当第三方收到数据，他需要使用发送者的公钥来验证JWT的签名部分。如果签名验证通过，即能够说明签名的数据必定来自对应的公钥，也就能够验证数据的发送者。
访问https://jwt.io/可以尝试这个签名和验证的过程，该平台的工具可以对数据进行自动签名。为了讲解的完整性，我们来看一下如何手动创建JWT。

### Web推送和JWT签名
JWT是一个字符串，主要有三部分组成，并由“.”连接起来。

第一和第二部分的字符串（JWT info和JWT data）是由JSON字符串经过base64编码而来，这意味着这两部分是明文可读的。
第一部分的字符串是关于JWT本身的信息，指明了创建签名的算法。we b推送使用的JWT必须包含一下信息：
```
{
  "typ": "JWT",
  "alg": "ES256"
}
```
第二部分是JWT的数据，这部分包含JWT的发送者，接收者和过期时间。对于Web推送，这部分数据必须包含一下信息：
```
{
  "aud": "https://some-push-service.org",
  "exp": "1469618703",
  "sub": "mailto:example@web-push-book.org"
}
```
`aud`字段是JWT的接收者，即JWT要发送给谁。对于Web推送来说接收者是推送服务，所以我们将其设置为推送服务的域名。
`exp`字段是JWT的过期时间，该字段可以阻止中间人截获JWT并重新使用。字段值是一个秒数的时间戳，且不能超过24小时。在Node.js中过期时间可以由一下代码生成：
```
Math.floor(Date.now() / 1000) + 12 * 60 * 60;
```
这里设置了12个小时而不是24小时，可以避免应用服务器和推送服务之间的时钟差异。
最后一个`sub`字段可以是一个URL或者mailto为开头的邮箱地址。该字段允许在必要的时候，推送想要联系到发送者就可以从JWT中找到发送的联系方式。这也是为什么web-push库需要一个邮箱地址。和第一部分一样，第二部分数据也必须是一个URL安全的base64字符串。
第三部分字符串是一个签名，将前两部分字符串用“.”连接之后组成的字符串称为“unsigned token”，然后对其进行签名。
签名的过程需要使用ES256（ES256签名算法的解释[点击这里](https://thiscute.world/posts/jwt-algorithm-key-generation/)）对为签名的token进行加密。根据[JWT规范](https://datatracker.ietf.org/doc/html/rfc7519)。ES256是“使用了P-256曲线加密算法和SHA-256哈希算法的ECDSA签名算法”的简称。使用crypto库创建签名的过程如下：
```
// Utility function for UTF-8 encoding a string to an ArrayBuffer.
const utf8Encoder = new TextEncoder('utf-8');

// The unsigned token is the concatenation of the URL-safe base64 encoded
// header and body.
const unsignedToken = .....;

// Sign the |unsignedToken| using ES256 (SHA-256 over ECDSA).
const key = {
  kty: 'EC',
  crv: 'P-256',
  x: window.uint8ArrayToBase64Url(
    applicationServerKeys.publicKey.subarray(1, 33)),
  y: window.uint8ArrayToBase64Url(
    applicationServerKeys.publicKey.subarray(33, 65)),
  d: window.uint8ArrayToBase64Url(applicationServerKeys.privateKey),
};

// Sign the |unsignedToken| with the server's private key to generate
// the signature.
return crypto.subtle.importKey('jwk', key, {
  name: 'ECDSA', namedCurve: 'P-256',
}, true, ['sign'])
.then((key) => {
  return crypto.subtle.sign({
    name: 'ECDSA',
    hash: {
      name: 'SHA-256',
    },
  }, key, utf8Encoder.encode(unsignedToken));
})
.then((signature) => {
  console.log('Signature: ', signature);
});
```
推送服务可以使用应用服务器密钥的公钥来解密这个签名，解密得出的字符串必须和未签名的字符串（即JWT的前两部分）相同。JWT必须放在请求头的`Authorization`字段中，并以WebPush为开头：
```
Authorization: 'WebPush [JWT Info].[JWT Data].[Signature]';
```
Web推送协议也规定了应用服务器密钥的公钥必须放在请求头的`Crypto-Key`字段中，公钥必须经过URL安全的base64编码，并以`p256ecdsa=`为开头。
```
Crypto-Key: p256ecdsa=[URL Safe Base64 Public Application Server Key]
```
### 加密推送数据
接下来我们来看一下如何使用消息推送发送数据，以便Web应用收到消息推送是，可以访问获取到的数据。
使用消息推送时，一个常见的问题是为什么发送给推送服务的数据必须是加密的。在原生APP中，推送的消息是明文的。Web推送的一个优势是所有的推送服务都遵守相同的规范使用相同的API，开发者不需要关注推送服务是谁，我们只要按照正确的格式向推送服务发送请求就能完成消息推送。不过这种模式也有一个缺点是开发者能会把消息发送给不安全的推送服务。通过对发送的数据进行加密，推送服务就无法获取到推送的数据，只有浏览器能够正常解密这些数据，从而保护用户数据。
推送数据加密的规范称为[Message Encryption spec](https://tools.ietf.org/html/draft-ietf-webpush-encryption)。
在讨论具体的消息加密步骤之前，我们先来回顾一下加密过程中所用到的一些技术。
### ECDH 和 HKDF
加密的整个过程中使用了ECDH和HKDF两种技术，他们为信息加密提供了很好的支持。

#### ECDH: 密钥交换（Elliptic Curve Diffie-Hellman key exchange）
想象一个场景，Alice和Bob想要共享一些信息，这两个人都有自己的公钥和私钥，并且都将自己的公钥公布给对方。ECDH的特点是Alice可以用自己的私钥和Bob的公钥创建一个密钥X，Bob也用自己的私钥和Alice的公钥独立的生成一个相同的密钥X。这样Alice和Bob只需要向对方传递自己的公钥，就能各自生成相同的密钥X，从而达到密钥X共享的目的。Alice和Bob可以使用X来加密或解密他们之间传输的数据。
ECHD依赖曲线的数学特性达到生成共享密钥的目的。ECHD更深入的介绍，我推荐可以看这个[视频的介绍](https://www.youtube.com/watch?v=F3zzNa42-tQ)。
代码实现方面，大多数的语言和平台都有库可以方便的生成这些密钥。在Node中我们使用以下方法：
```
const keyCurve = crypto.createECDH('prime256v1');
keyCurve.generateKeys();

const publicKey = keyCurve.getPublicKey();
const privateKey = keyCurve.getPrivateKey();
```
#### HKDF 基于HMAC的密钥生成函数（HMAC based key derivation function）
[HKDF规范](https://tools.ietf.org/html/rfc5869)中对该加密技术有比较简明描述。HKDF是基于HMAC密钥派生函数，它可以将一个弱密码转化为密码学的强密码，例如它可以用来将上一步的密钥交换算法生成的共享密码转化为适合加密、完整性校验或认证的强密钥。本质上讲，HKDF能将不是特别安全的密钥变得更加安全。
Web推送协议规范要求加密必须使用SHA-256作为哈希算法，所以HKDF生成的结果密钥长度不能超过256位（32字节）。在Node中的实现如下：
```
// Simplified HKDF, returning keys up to 32 bytes long
function hkdf(salt, ikm, info, length) {
  // Extract
  const keyHmac = crypto.createHmac('sha256', salt);
  keyHmac.update(ikm);
  const key = keyHmac.digest();

  // Expand
  const infoHmac = crypto.createHmac('sha256', key);
  infoHmac.update(info);

  // A one byte long buffer containing only 0x01
  const ONE_BUFFER = new Buffer(1).fill(1);
  infoHmac.update(ONE_BUFFER);

  return infoHmac.digest().slice(0, length);
}
```
简单的概括一下ECDH和HKDF：
ECDH通过共享通信双方的公钥来生成一个共享的密钥。HKDF将不够安全的密钥变得更加安全。
以上的这些会在加密推送数据的时候用到，接下来我们来看一下我们所需要的输入以及数据是如何加密。
#### 输入
当我们发送包含数据的消息推送给用户时，我们需要提供三个输入项：
1. 数据本身
2. `PushSubscription`订阅对象的`auth`密钥
3. `PushSubscription`订阅对象的`p256dh`密钥

我们已经知道auth和p256dh的值可以从PushSubscription中获取，我们可以通过一下方式获取：
```
subscription.toJSON().keys.auth;
subscription.toJSON().keys.p256dh;

subscription.getKey('auth');
subscription.getKey('p256dh');
```
auth的值应该被保存在安全的地方，不能分享给其他应用。p256dh时一个公钥，可以将它视为客户端浏览器的公钥，订阅对象的公钥是由浏览器生成的，浏览器的私钥时私有的，我们无法知道。浏览器会用它解密接收到的数据。
##### Salt
Salt值时一个16字节的随机数据，在NodeJS中使用以下方式生成Salt值：
```
const salt = crypto.randomBytes(16);
```
##### 公钥/私钥
公钥和私钥需要使用P-256椭圆曲线算法生成，在NodeJS中的代码实现如下：
```
const localKeysCurve = crypto.createECDH('prime256v1');
localKeysCurve.generateKeys();

const localPublicKey = localKeysCurve.getPublicKey();
const localPrivateKey = localKeysCurve.getPrivateKey();
```
我们可以称这个密钥是“本地密钥”，他们是用来加密要发送的数据的，和之前提到的应用服务器公钥、私钥密钥对无关。
当发送推送数据时，我们有auth密钥、订阅公钥（p256dh）、新生成的salt值和本地密钥对，加密所需要的准备工作就已经完成。
#### 共享密钥
加密的第一步是使用订阅对象的公钥，和我们新生成的私钥（就像上文提到的Alice和Bob的例子）生成共享密钥。
```
const sharedSecret = localKeysCurve.computeSecret(
  subscription.keys.p256dh,
  'base64',
);
```
下一步需要使用这个共享的密钥生成一个伪随机密钥。（PRK）
#### 伪随机密钥（Pseudo random key）
伪随机密钥（PRK）是auth密钥和上一步生成的共享密钥的组合。
```
const authEncBuff = new Buffer('Content-Encoding: auth\0', 'utf8');
const prk = hkdf(subscription.keys.auth, sharedSecret, authEncBuff, 32);
```
你可能会有疑惑`Content-Encoding: auth\0`字符串的作用是什么。简单的讲，该字符串没有明确的目的，浏览器在解密收到的消息时，会尝试查找Content-Encoding字符串。\0会在Buffer的末尾增加一个字节的数据，并用0值填充。解密数据时，浏览器期望的数据是content encoding后面有0填充的一个字节，然后是要发送加密数据。
我们的伪随机密钥是使用auth、共享密钥、和一小段编码字符串通过HKDF生成的。
#### 上下文
上下文是一个字节集合，浏览器可以使用它来解密收到的数据。它本质上是个字节数组，包含了订阅的公钥“本地密钥”的公钥。（加密时候使用本地密钥的私钥和订阅的公钥，为了能让浏览器解密该数据，需要提供本地密钥的公钥和订阅的公钥。因为根据订阅的公钥，浏览器可以找到订阅私钥）
```
const keyLabel = new Buffer('P-256\0', 'utf8');

// Convert subscription public key into a buffer.
const subscriptionPubKey = new Buffer(subscription.keys.p256dh, 'base64');

const subscriptionPubKeyLength = new Uint8Array(2);
subscriptionPubKeyLength[0] = 0;
subscriptionPubKeyLength[1] = subscriptionPubKey.length;

const localPublicKeyLength = new Uint8Array(2);
subscriptionPubKeyLength[0] = 0;
subscriptionPubKeyLength[1] = localPublicKey.length;

const contextBuffer = Buffer.concat([
  keyLabel,
  subscriptionPubKeyLength.buffer,
  subscriptionPubKey,
  localPublicKeyLength.buffer,
  localPublicKey,
]);
```
上下文buffer包含一个`P-256\0`标签，然后是订阅公钥的字节长度值和公钥本身，再后面的本地公钥和字节长度和本地公钥本身。
上下文的值用来创建一次性密钥（nonce）和内容加密密钥。（content encryption key，CEK）
#### 内容加密密钥和一次性密钥
[一次性密钥（nonce）](https://en.wikipedia.org/wiki/Cryptographic_nonce)可以阻止重放攻击，因为它只能使用一次。内容加密密钥（CEK）是我们用来加密发送数据的最终密钥。首先我们需要为nonce和CEK创建字节数据，他们是由内容编码字符串和上一步的上下文组成：
```
const nonceEncBuffer = new Buffer('Content-Encoding: nonce\0', 'utf8');
const nonceInfo = Buffer.concat([nonceEncBuffer, contextBuffer]);

const cekEncBuffer = new Buffer('Content-Encoding: aesgcm\0');
const cekInfo = Buffer.concat([cekEncBuffer, contextBuffer]);
```
这两个值的生成需要使用salt值、伪随机密钥（PRK）、nonceInfo和cekInfo生成：
```
// The nonce should be 12 bytes long
const nonce = hkdf(salt, prk, nonceInfo, 12);

// The CEK should be 16 bytes long
const contentEncryptionKey = hkdf(salt, prk, cekInfo, 16);
```
然后我们就得到了一次性密钥和内容加密密钥。
#### 执行加密
现在有了内容加密密钥，我们可以加密数据了。我们使用内容加密密钥和一次性密钥作为初始化向量创建一个AES128（一种对称加密算法）密码，在Node中代码如下：
```
const cipher = crypto.createCipheriv(
  'id-aes128-GCM',
  contentEncryptionKey,
  nonce,
);
```
加密数据之前我们还需要定义padding值，放在要发送的数据之前。原因是这个padding值可以阻止中间人通过数据的长度来猜出消息数据的类型。
你必须添加两个字节指定padding值的长度。例如你不想添加padding值，前两个字节的值必须是0，这两个字节之后的数据就是要发送的数据。如果你想添加5字节的padding值，那么前两个字节的值就是5，然后数据消费者在读取数据时，会向后再偏移五个字节。
```
const padding = new Buffer(2 + paddingLength);
// The buffer must be only zeros, except the length
padding.fill(0);
padding.writeUInt16BE(paddingLength, 0);
```
接下来使用加密函数对padding值和有效载荷进行加密
```
const result = cipher.update(Buffer.concat(padding, payload));
cipher.final();

// Append the auth tag to the result -
// https://nodejs.org/api/crypto.html#crypto_cipher_getauthtag
const encryptedPayload = Buffer.concat([result, cipher.getAuthTag()]);
```
终于我们有了经过加密的数据了。剩下的工作就是如何把加密的数据发送给推送服务。
#### 加密数据和请求头
为了能向推送服务发送加密的数据，我们需要再POST请求中定义一些不同的请求头字段。
##### Encryption字段
请求头的Encryption字段必须包含salt加密数据时候用到的slat值。
salt值的长度时16字节，必须经过base64 URL编码：
```
Encryption: salt=[URL Safe Base64 Encoded Salt]
```
##### Crypto-Key字段
在之前的应用服务器密钥章节，我们已经看到`Crypto-Key`字段包含了应用服务器密钥的公钥，不仅如此，该字段还用来传递前面提到的本地密钥对的公钥。这个公钥用来加密要发送的数据。该字段完整形式如下：
```
Crypto-Key: dh=[URL Safe Base64 Encoded Local Public Key String]; p256ecdsa=[URL Safe Base64 Encoded Public Application Server Key]
```
##### Content type, length & encoding字段
`Content-Length`字段指经过加密后数据的字节数，`Content-Type`和`Content-Encoding`字段的值是固定的，三个字段形式如下：
```
Content-Length: [Number of Bytes in Encrypted Payload]
Content-Type: 'application/octet-stream'
Content-Encoding: 'aesgcm'
```
设置了请求头字段，我们需要把经过加密的数据作为请求的请求体发送。注意`Content-Type`的值是`application/octet-stream`。这是因为加密数据必须通过字节流的形式发送。在NodeJS中的实现如下：
```
const pushRequest = https.request(httpsOptions, function(pushResponse) {
pushRequest.write(encryptedPayload);
pushRequest.end();
```
##### 其他字段
我们已经讲解了用于传递JWT、应用服务器密钥的请求头，他们可以帮助推送服务识别发送请求的应用。还有其他与数据加密相关的请求头。还有一些额外的头字段，推送服务用他们来修改消息发送的行为，有些字段是必须的，有些则是可选的。
##### TTL头字段（必须的）
TTL字段（time to live）的值是个秒数的整数值，该字段指定推送服务在分发消息前，消息在推送服务中的生存期。当TTL过期时，该消息将会被推送服务移除，并不再发送该消息。
```
TTL: [Time to live in seconds]
```
如果将TTL的值设为0，推送服务会立刻分发该消息，但是如果消息没有发送到用户的设备，该消息会从推送服务的队列中立刻被删除。技术上讲，推送服务可以减少推送消息的TTL值。不过你可以通过检查推送服务响应头肿的TTL字段知道该字段有没有被修改。
##### Topic头字段（可选）
Topic字段的值是个字符串，可以用新的消息替换还未发送的消息，但是两个消息需要用相同的Topic值。该功能在某些场景下是很有用的，例如你有多个消息需要发送，但是某些设备还是离线状态，但是当用户的设备恢复在线之后，你只想让用户看到最近的一条消息。
##### Urgency字段（可选）
Urgency字段告诉推送服务消息对于用户的重要性。该字段可以使推送服务帮助用户节省设备电量，尤其是当用户设备处于低电量的时候，设备只有在收到重要的消息时才会被唤醒。该字段的默认值时normal，其他可选值如下：
```
Urgency: [very-low | low | normal | high]
```
如果你还有关于Web Push协议其他的问题，想了解这个过程的其他细节，可以参考[官方库（web-push-libs）](https://github.com/web-push-libs)是如何触发消息推送的。只要有了经过加密的数据和上面所需的请求头，你可以使用`PushSubscription`对象向`endpoint`发送post请求。那么我们如何处理POST请求的响应呢？
#### 处理推送服务的响应
向推送服务发送请求后，你需要检查响应的状态码，他可以告诉你请求是否成功。

| 状态吗 |描述  |
| --- | --- |
|201  | 创建成功，推送服务收到请求并会发送该消息 |
|429  | 请求过于频繁，该状态码说明你的应用服务器向推送服务发送请求过于频繁，收到该状态码后，通常需要检查`Retry-After`字段，它指定了需要在多久后才能再次向推送服务发送请求 |
|400  | 请求验证失败，该状态码说明请求头字段验证失败，或者个事不正确 |
|404  | Not Found，该状态码说明订阅对象已经过期，不再可用。此时你需要删除订阅对象，并在客户端重新订阅 |
|410  | 该状态码说明订阅对象不再有效，此时需要删除订阅对象。当用户调用`unsubscribe()`方法时，会收到该状态码 |
|413  |发送的数据体积过大，推送服务支持的数据为4KB |
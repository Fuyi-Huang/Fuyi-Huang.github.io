---
title: leetcode Combinations 刷题总结
tags: 
    - programming 
    - error
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
mathjax: true
mathjax_autoNumber: true
---


昨天在公司快下班的时候，突然收到线上的告警信息，线上其中一台机器请求渠道的服务失败，我们的后端代码是PHP，上到机器上面查看的时候发现代码记录的失败日志是：

```text
request failed,curl occur an error:
SSL certificate problem: unable to get local issuer certificate
Original verbose string:
*   Trying *.*.*.*...
* TCP_NODELAY set
* Connected to *.*.* (*.*.*.*) port 443 (#0)
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* SSL certificate problem: unable to get local issuer certificate
* stopped the pause stream!
* Closing connection 0
```
为了安全性，*Trying*和*Connected to*后面的域名和IP信息隐去了，以下的也会这样。

很直观，这是SSL证书问题，这个问题从来没遇到过。我将错误信息*unable to get local issuer certificate*在网上查发现可以通过下面的方法解决

1. 下载最新的CA包，里面包含了现在最新的可信任的CA根服务商，[下载地址https://curl.haxx.se/docs/caextract.html](https://curl.haxx.se/docs/caextract.html)
2. 在php.ini指定php curl扩展的ca证书文件
    ```ini
    curl.cainfo="/etc/ssl/certs/cacert.pem"
    openssl.cafile="/etc/ssl/certs/cacert.pem"
    ```
    或者在代码里面用curl请求时，指定
    ```php
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    // or 
    $certificate = "/etc/ssl/certs/cacert.pem";
    curl_setopt($ch, CURLOPT_CAINFO, $certificate);
    curl_setopt($ch, CURLOPT_CAPATH, $certificate);
    ```
3. 重启php服务

因为是最后一天上班了，是不允许再修改线上代码了，或者修改php.ini配置从而影响机器上的所有服务。只能找别的办法。


但是很奇怪的是，只有其中一台机器会这样，线上的其他机器看日志没有这个问题，部署的代码都是一样的啊。所以我猜想是机器问题，只是这台机器出了问题，所以我在机器上用curl命令访问渠道的链接，在正常和有问题的机器上的结果分别是：

```text
* About to connect() to *.*.* port 443 (#0)
*   Trying *.*.*.*... connected
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS handshake, Server key exchange (12):
* SSLv3, TLS handshake, Server finished (14):
* SSLv3, TLS handshake, Client key exchange (16):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSL connection using ECDHE-RSA-AES128-GCM-SHA256
```

```text
* About to connect() to *.*.* port 443 (#0)
*   Trying *.*.*.*... connected
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS alert, Server hello (2):
* SSL certificate problem, verify that the CA cert is OK. Details:
error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed
* Closing connection #0
curl: (60) SSL certificate problem, verify that the CA cert is OK. Details:
error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
```

第二个是有问题的机器的请求，果然也是证书的问题。最后实在搞不明白为什么突然就会有证书的问题。后来，问了渠道之后，知道当时渠道他们换了他们的证书，他们的证书颁发机构(CA)也换了，然后了解到有信任证书颁发机构这一回事[https://curl.haxx.se/docs/sslcerts.html](https://curl.haxx.se/docs/sslcerts.html)。（具体的TLS握手过程请看下面的相关知识）
现在事情就逐渐明了了，因为渠道换了证书颁发机构，同时新的证书颁发机构不在那台问题机器的信任列表里（后面偶然查到那台机器比其他机器都要老，是16年以前的）。所以解决办法是把新的证书颁发机构的根证书下载，并放在 **/etc/ssl/certs/** 目录下，该目录是php和curl等使用openssl的都是默认的这个路径作为证书存放地址。

但是，直接放在/etc/ssl/certs/目录下是无法被使用的，因为该目录下的证书的命名必须是：以证书哈希为文件名，.0 为文件后缀
下面是生成证书哈希名的方法（我偶然在一个文档看到的[https://www.php.net/manual/en/openssl.configuration.php#122682](https://www.php.net/manual/en/openssl.configuration.php#122682)）
```php
<?php
$path = "/path/to/cacert.pem";
$parsedCertificatedObject = openssl_x509_parse(file_get_contents($path));
echo $parsedCertificatedObject['hash'];

```

这样把*证书哈希.0*文件放到 **/etc/ssl/certs/** 目录下问题就解决了。

## 相关知识

[TLS握手流程](https://en.wikipedia.org/wiki/Transport_Layer_Security#Protocol_details)：

TLS握手主要包括两种，基本的，和包含客户端证书校验的双向校验握手，下面讲解基本的

1. 客户端发送ClientHello包，声明客户端支持的最高TLS版本，一个随机数，支持的一系列加密算法和压缩算法。

2. 服务端发送ServerHello包，包含选择的TLS版本，一个随机数，从客户端支持的加密算法和压缩算法中各确认一个。

3. 服务端发送他的证书给客户端，这一步客户端会校验服务端发送过来的证书是否是可信任的证书

4. 对于选择DHE, DH_anon系列的加密算法，服务端会发送ServerKeyExchange包

5. 服务端发送ServerHelloDone包，表示他已完成握手协商

6. 客户端发送ClientKeyExchange包，包含PreMasterSecret，客户端的公钥（如果没有会现生成一对公私钥），或者什么也不发送（根据选择的加密算法而异）。PreMasterSecret会用服务端的公钥加密，这样只有服务端的私钥可以解密出来。

7. 客户端和服务端使用之前的随机数和PreMasterSecret生成一个公共的密钥，叫"master secret"。之后同一个连接中的所有数据都会用这个master secret用相应的对称加密算法加密传输内容。

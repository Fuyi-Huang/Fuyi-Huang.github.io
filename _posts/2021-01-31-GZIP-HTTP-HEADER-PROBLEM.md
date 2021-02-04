---
title: 记一次和gzip有关的java OKHTTP的坑
tags: 
    - programming 
    - error
    - Java
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
mathjax: true
mathjax_autoNumber: true
---

事情的起因是，有一个新项目，需要访问内网的服务A，服务A其实本身不提供服务的实现，只是做请求的转发，实际服务的实现是服务B。服务A接收到请求，转给服务B，得到服务B的响应之后输出。
我们之前也有用过服务A，是用在PHP项目上，现在的新项目是Java项目。

因为之前接过服务A，所以以为会很快接入，没想到掉进了个坑。

Java里面请求服务A用的是HTTP client是OKHttp3，请求得到服务A的响应没办法解json，输出之后发现是一堆的乱码，打印了响应的HTTP头信息发现Content-Length比预期小很多。
然后，用之前有问题的请求体curl手动调用服务A，发现响应是正常的。而且之前PHP项目也一直在线上跑着，没出现过类似的问题。

然后问负责服务A的同事，他说他之前遇到过，不过不是使用Java项目遇到的，是使用postman请求他自己的服务A，发现得到的响应也是乱码，但是手动curl也是没问题的。我自己用postman试了下，发现也是。
我自己用postman试着跳过服务A直接访问服务B，发现没有问题的，响应正常。对比服务A和服务B相同请求的响应发现，服务A的响应HTTP头特别少，只有Content-Lenght，一眼看到服务B的响应头里面有Content-Encoding: gzip。
立马想到，会不会是服务A的响应是GZIP压缩过的数据，所以Content-Length数值才会比预期小，且是乱码。立马在Java项目里把http的响应在进行json解析之前先进行gzip解压，发现果然没问题，反复请求了好多次，都没问题。
赶快完成需求，因为这个耽误了不少的时间，接下来验收、测试、上线都没问题。

虽然是赶在deadline之前上线了，但是之前PHP和curl没问题，但是postman和Java请求就有问题，上线之后一直担心哪里有坑。第二天花了一些时间专门研究下，我先是在机器上用`tcpdump host {服务A IP} -s 1024 -l -A`命令监控服务A的流量，
然后运行Java项目，观察Java项目请求服务A的流量，和用curl手动请求服务A流量。
![image-title-here](assets/images/2021-01-31/1.png){:class="img-responsive"}
![image-title-here](assets/images/2021-01-31/2.png){:class="img-responsive"}
看上图1curl的请求和图2Java项目的请求，发现Java项目的请求里面有`Accept-Encoding: gzip`请求头，但是curl的流量里面没有这个请求头。而且我请求服务A的时候没有自己设置这个请求头，可想而知是OKHTTP自动设置的。
看下面OKHTTP BridgeInterceptor类的源码
```java
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    // 就是这里，对于没有设置Accept-Encoding和Range请求头，自动设置一个Accept-Encoding: gzip的请求头
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 然后在这里判断如果是自动设置的gzip，且响应头里面有Content-Encoding: gzip，则做gzip解压
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like {@code a=b; c=d}. */
  private String cookieHeader(List<Cookie> cookies) {
    StringBuilder cookieHeader = new StringBuilder();
    for (int i = 0, size = cookies.size(); i < size; i++) {
      if (i > 0) {
        cookieHeader.append("; ");
      }
      Cookie cookie = cookies.get(i);
      cookieHeader.append(cookie.name()).append('=').append(cookie.value());
    }
    return cookieHeader.toString();
  }
}
```
源码里面，如果请求没有设置Accept-Encoding头，则会默认设置Accept-Encoding: gzip，然后在响应里面判断Content-Encoding是gzip的话，则做gzip解压再返回给调用方。
一般来说，HTTP请求使用gzip压缩可以大大提升效率。但是，我们遇到了一个不称职的请求代理*服务A*，对于请求代理而言，服务A是不称职的，他会把请求调用方的http header透传给
服务B，但是他会却把服务B响应的http header都丢弃了。响应里面没有`Content-Encoding: gzip`，导致OK HTTP BridgeInterceptor里面不会对响应做gzip解压再返回给调用方，
而是直接把gzip的压缩数据返回给调用方，这导致了我们一开始说的响应乱码问题。而使用curl和PHP里面用curl库不是自动设置Accept-Encoding，没有设置的话，服务B响应不会返回压缩数据，
从服务A得到的响应也不会是压缩过的，所以一直没问题。

### 总结

如果没有特别设置Accept-Encoding，Java OKHTTP库会默认设置`Accept-Encoding: gzip`，然后如果响应里面判断有`Content-Encoding: gzip`则把数据做gzip解压之后再返回给调用方。
如果OKHTTP自动设置了`Accept-Encoding: gzip`，响应数据也是gzip压缩的，但是没有`Content-Encoding: gzip`头，则是把gzip压缩数据返回给调用方。

### 补充知识

监听特定IP的http流量: `tcpdump host {IP} -s 1024 -l -A`

[Accept-Encoding](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Encoding)

[Content-Encoding](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Encoding)
title: okHttp3源码解析(三)-CallServerInterceptor
date: 2018-08-29 00:00:00
categories:
- 源码解析
tags:
- 源码解析
- okHttp3
---
## 前言
本篇主要看下`CallServerInterceptor`, 关于他在整个请求中起到的作用, okHttp已经告诉我们, 可以看出它作为责任链中的最后一个环节, 承担了对服务端进行请求的工作.
> This is the last interceptor in the chain. It makes a network call to the server.
<!-- more -->
## 正文
`okHttp`对于对服务钱的请求与相应, 底层都是通过`okio`对`socket`进行操作.
老样子, 我们直接上代码
``` java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    // 在 ConnectInterceptor创建
    HttpCodec httpCodec = realChain.httpStream();
    // 在 RetryAndFollowUpInterceptor创建
    StreamAllocation streamAllocation = realChain.streamAllocation();
    // 在 ConnectInterceptor获取
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();
    // 发送请求时间戳为当前时间戳
    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    // 发送请求头, 通过Okio
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    // GET or HEAD 不需要
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        // 请求刷新, okio处理
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        // 构建Response.Builder, 当response状态为100, 则返回null
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // head成功响应的情况下
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        // 请求体的输出流
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        // 发送请求体
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // HTTP/1请求协议, 而且初次握手失败
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        // 禁止同主机请求新流的分配
        streamAllocation.noNewStreams();
      }
    }

    // flush
    httpCodec.finishRequest();
    // 如果是GET请求, 或者需要'100- continue'握手成功的情况下
    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      // 构建responseBuilder
      responseBuilder = httpCodec.readResponseHeaders(false);
    }
    // 获取响应
    Response response = responseBuilder
        // 原请求
        .request(request)
        // 握手情况
        .handshake(streamAllocation.connection().handshake())
        // 请求时间
        .sentRequestAtMillis(sentRequestMillis)
        // 响应时间
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      // 即使我们没有请求, 服务端也会发送一个100-continue
      // 重新读取真正的响应
      responseBuilder = httpCodec.readResponseHeaders(false);
      // 构建response
      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      // 我们需要确保不会反悔一个空的响应体
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    // 如果请求关闭连接, 则关闭
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }
    // 抛出协议异常
    // 204: No Content
    // 205: Reset Content
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```
### 请求
我们来一步步看下代码
``` java
RealInterceptorChain realChain = (RealInterceptorChain) chain;
    // 在 ConnectInterceptor创建
    HttpCodec httpCodec = realChain.httpStream();
    // 在 RetryAndFollowUpInterceptor创建
    StreamAllocation streamAllocation = realChain.streamAllocation();
    // 在 ConnectInterceptor获取
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();
```
在`CallServerInterceptor`在, 核心工具类就是`HttpCodec`, 他的初始化我们在上篇[ConnectInterceptor解析](https://xiaozhuanlan.com/topic/5208976413)中可以看到.这里的步骤其实就是工具的准备.
``` java
httpCodec.writeRequestHeaders(request);
```
我们以HTTP/1.1协议来看, 那么具体要看`Http1Codec`中的实现
``` java
@Override public void writeRequestHeaders(Request request) throws IOException {
    // requestLine 就是我们请求报文内容的首行, 譬如 "GET / HTTP/1.1"
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);
  }

  final BufferedSink sink;
  public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
  }
```
可以看到在`writeRequest`中的方法, 就是针对`BufferedSink`对象的写操作, 在上篇`ConnectInterceptor`中进行三次握手连接的时候, 会进行初始化的工作, 我们会发现他是针对`Socket`的包装, 可以看做是`Socket`的输出流, 所以这里相当于是`Socket`的写入动作, 可以看出来, 这里会请求发送`header`.
``` java
if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        // 请求刷新, okio处理
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        // 构建Response.Builder, 当response状态为100, 则返回null
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // head成功响应的情况下
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        // 请求体的输出流
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        // 发送请求体
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // HTTP/1请求协议, 而且初次握手失败
        // 禁止同主机请求新流的分配
        streamAllocation.noNewStreams();
      }
    }
```
当请求首部字段包含`Expect:100-continue`, 一般在请求上传大容量body或者是需要验证的时候, 这样避免大文件传送失败带来的带宽浪费。所以需要判断请求体不为空的情况下, 并且请求首部包含该字段的情况下, 刷新请求, 构建`responseBuilder`对象, 如果这个时候服务端响应成功, 则`responseBuilder`对象为null, 并进行body的请求;而如果第一次请求头响应失败的情况下, 那么如果是HTTP/1的请求协议下, 就会禁止同host的请求新流的分配.
### 响应
以上就是请求的过程, 然后看响应
``` java
// 如果是GET请求, 或者需要'100- continue'握手成功的情况下
    if (responseBuilder == null) {
      // 构建responseBuilder
      responseBuilder = httpCodec.readResponseHeaders(false);
    }
    // 获取响应
    Response response = responseBuilder
        // 原请求
        .request(request)
        // 握手情况
        .handshake(streamAllocation.connection().handshake())
        // 请求时间
        .sentRequestAtMillis(sentRequestMillis)
        // 响应时间
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
```
在请求成功的情况下, 会重新构建`responseBuilder`对象, 通过它来构建响应报文.
``` java
int code = response.code();
    if (code == 100) {
      // 即使我们没有请求, 服务端也会发送一个100-continue
      // 重新读取真正的响应
      responseBuilder = httpCodec.readResponseHeaders(false);
      // 构建response
      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }
```
如果客户端响应`100`状态码, 这个时候, 我们就需要重新读取获取真正的内容响应.

剩下的代码就是一些异常的处理, 这里就不做分析了.

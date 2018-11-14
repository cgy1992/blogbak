title: okHttp3源码解析(一)
date: 2018-08-20 00:00:00
categories:
- 源码解析
tags:
- 源码解析
- okHttp3
---

> 源码基于3.11.0版本

okHttp的请求分为两种, 同步和异步的.
本篇主要了解下两种请求的请求流程, 差异.
<!-- more -->
## 同步请求
我们先看下同步请求api的使用
``` kotlin
val okHttpClient by lazy { OkHttpClient() }
private fun synchronousRun(url: String): String?{
        val request = Request.Builder()
                .url(url)
                .build()
        val response = okHttpClient.newCall(request).execute()
        return response?.body()?.string()
    }
```
### Request
`Request`的代码就不看了, 可以看出是使用建造者模式, 根据具体配置去build.要注意的是, 这里传入的url, 如果是websocket协议的url, 会替换成http, 最后url会包装成一个`HttpUrl`对象
``` java
public Builder url(String url) {
      if (url == null) throw new NullPointerException("url == null");

      // Silently replace web socket URLs with HTTP URLs.
      if (url.regionMatches(true, 0, "ws:", 0, 3)) {
        url = "http:" + url.substring(3);
      } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
        url = "https:" + url.substring(4);
      }

      return url(HttpUrl.get(url));
    }
```
另外关于`GET`的请求, 在`Request`构造器初始的时候, 就会默认为`GET`请求, 所以如果是`POST`请求的时候, 需要调用`Request.Build().post(body)`方法
``` java
public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
public Builder post(RequestBody body) {
  return method("POST", body);
}
```
![img](./requestbuilder.png)

我们可以看下Request可配置项, 分别包含一个Url, 一个请求的方法, header列表, 请求体和tags(关于tags我们后续再讲)
### Call
请求需要准备一个`Call`对象, 他表示在未来的时间点内可以随时执行准备好的请求. 我们可以看到实际执行的时候, 其实使用的是`RealCall`对象, 具体看下请求执行的过程
``` java
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
### execute
``` java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    // 创建一个跟踪当前执行方法的堆栈, 赋值在失败重试的拦截器中
    captureCallStackTrace();
    // 监听
    eventListener.callStart(this);
    try {
      // 通过dispatcher做管理, 将call加入同步请求队列中
      client.dispatcher().executed(this);
      // 获取返回的结果, 并执行一系列拦截器处理
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      // 通过dispatcher通知结束
      client.dispatcher().finished(this);
    }
  }
```
`executed`表示对应的`call`是否已经执行, 这里同步锁可以避免了竞态条件的出现, 可以看出一个`call`实例只能被执行一次, 是一个"消耗品"
关于`Dispatcher`我们可以看下在同步请求下他具体的执行和结束的代码
``` java
// 执行方法
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
// 通知结束
void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      // 移除call
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
    // 判断当前异步请求和同步请求数总和是否为0, 如果为0 则调用闲置时候的回调
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
```
可以看到在同步请求的时候, `dispatcher`的执行和完成通知, 实际是针对于`runningSyncCalls`对象的管理.
``` java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 自定义拦截器
    interceptors.addAll(client.interceptors());
    // 失败重试或者重定向的拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    // 请求和响应的转换处理拦截器
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 缓存拦截器, 从缓存中请求并将响应写入缓存
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 建立连接拦截器, 并继续下一个拦截器
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      // 用户自定义的拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    // 最后一个拦截器, 处理网络调用服务器
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
我们可以看到, 实际执行是`RealInterceptorChain.proceed`
``` java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // 请求下一个责任链
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    // 获取当前的拦截器
    Interceptor interceptor = interceptors.get(index);
    // 执行, 返回响应
    Response response = interceptor.intercept(next);

    // 保证下一个拦截器会调用到chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // 保证拦截器返回的响应不为空
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }
    // 保证响应的body不为空
    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```
这里可以把拦截器理解成工厂模式, 包装`Request`, 递归执行拦截器抽象的`intercept`方法, 然后将返回的`Response`再传回上一个拦截器内做处理.再返回看前面的`getResponseWithInterceptorChain`方法, 可以看出真正请求执行的就在这一块, 根据责任链的设计思想, 将操作分开进行处理.
具体流程可看下图
![img](./chain.jpg)
## 异步请求
现在我们再回头看下异步请求时, 与同步请求有什么区别.
``` java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
在执行异步调用的时候, okHttp包装了一个`AsyncCall`对象通过`dispatcher`进行执行.`AsyncCall`继承`NamedRunnable`, 实现了`Runnable`接口.
``` java
synchronized void enqueue(AsyncCall call) {
  // 判断当前异步请求数是否小于最大请求数 以及 同主机的请求数是否小于每个主机的最大请求数
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```
可以看到异步请求, 在`dispatcher`内部会维护两个集合以及一个线程池: `runningAsyncCalls`表示当前执行的异步请求队列, `readyAsyncCalls`表示等待执行的异步请求队列.执行内容我们可以看`AsyncCall.execute`, 他在判断请求是否取消后, 会调用`getResponseWithInterceptorChain`方法, 后面的请求走到的步骤就与同步相同了.然后通过`dispatcher`关闭管理.
``` java
@Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 责任链执行
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

``` java
private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```
这里唯一的区别是, 异步请求调到`Dispatcher.finished`的时候, 会调到`promoteCalls`方法, 他用来判断调度当前异步请求数是否超过最大请求, 如果没有, 则会从异步请求等待队列中获取出来再进行请求执行.由此, `Dispatch`才做到了内部针对于异步请求的线程管理, 实现了对应策略下同时请求的最大化.
## 总结
本篇主要了解同步请求和异步请求的主干流程, 可以看出异步请求和同步请求的区别, 就在于, 异步请求真正执行是通过`Dispatcher`进行管理与执行, 虽然同步请求也用到了`Dispatcher`, 但它主要是用来做同步请求队列的管理.两类请求真正的请求网络的处理, 都是通过调用`getResponseWithInterceptorChain`方法进行处理.
整体流程可以参照[拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/#section-2)里的下图
![img](./okhttp_full_process.png)

title: okHttp3源码解析(二)-ConnectInterceptor
date: 2018-08-24 00:00:00
categories:
- 源码解析
tags:
- 源码解析
- okHttp3
---
## 前言
上文简单概括了下`okHttp3`请求的整体流程, 本篇主要看下`ConnectInterceptor`的主要工作内容
<!-- more -->
## 正文
已知拦截器链都是从各拦截器的`intercept`方法开始调用, 那么我们从`ConnectInterceptor`的`intercept`代码开始看起
``` java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    // 从RetryAndFollowUpInterceptor获取
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // 判断请求是不是GET方法, 不是的情况下,需要进行有效监测
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 新建HttpCodec
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
我们可以看到, 获取连接的拦截器内, 主要只有三个步骤:
1. 初始化`HttpCodec`
2. 通过`streamAllocation`获取连接
3. 将`httpCodec`和`connection`作为参数带到下个拦截器的调用方法中

这里`HttpCodec`我们可以大概了解下, 它是个抽象类, 有`Http1Codec`和`Http2Codec`实现它, 分别根据Http/1.1,和Http/2做针对请求响应不同的编解码处理.

而`StreamAllocation`对象是在`RetryAndFollowUpInterceptor`中新建获取到的, 它做了`Streams`, `Connections`, `Calls`的关系管理.这里要注意的是`Streams`表示的是逻辑层面的连接(流), 每个连接(`Connection`)都定义了可以并发请求的连接(`Streams`), HTTP/1.x每次只能携带一次, HTTP/2可以携带多次.

回头我们看下`streamAllocation.newStream`做了什么
``` java
public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      // 遍历查找健康可用的连接
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      // HttpCodec初始化
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```
可以看到它这里也就做了三个动作
1. 配置连接超时, 读取超时, 写超时的时间.
2. 查找健康可用的连接
3. 根据可用连接初始化`HttpCodec`

继续往下看:
``` java
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      // 查找连接, 更加倾向连接池内已存在的连接, 否则会重新构建
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      // 如果是全新的连接, 则跳过可用检查, 直接返回
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      // 判断是否是可用连接
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        // 禁止新流创建
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```
1. 循环查找连接
2. 如果是全新的连接, 则跳过检查, 直接返回
3. 判断是否可用连接, 如果不是, 则禁止新流创建

继续看`findConnection`方法
``` java
/**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    // 从连接池中找到连接
    boolean foundPooledConnection = false;
    // 实际需要返回的连接
    RealConnection result = null;
    // 对应找到的路由
    Route selectedRoute = null;
    // 对应可释放的连接
    Connection releasedConnection;
    // 需要关闭的socket
    Socket toClose;
    synchronized (connectionPool) {
      // 异常判断
      // 判断是否连接已经被释放, codec是否为空, 请求是否被取消
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // 尝试寻找已经存在的连接来使用.
      // 但是需要注意的是, 已存在的连接可能已经无法再创建新的流
      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      // toClose如果无法创建流, 需要关闭的socket
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        // 如果当前连接不为空, 就说明这个连接是可以用的
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }
      // 如果没有现成的连接
      if (result == null) {
        // Attempt to get a connection from the pool.
        // 尝试从连接池中获取
        Internal.instance.get(connectionPool, address, this, null);
        // 如果有复用的连接
        if (connection != null) {
          // 表示找到连接池可复用的连接
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    // 关闭socket
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      // 如果有存在已分配的连接或者是连接池内可复用的连接, 则直接返回该连接对象
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      // 切换路由
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          // 获取可复用的连接
          Internal.instance.get(connectionPool, address, this, route);
          // 如果存在可复用连接
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }
      // 如果没有找到可复用的连接
      if (!foundPooledConnection) {
        // 如果当前路由为空
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        // 创建新的连接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    // 如果第二次有找到, 则返回复用的连接
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    // 做三次握手
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    // 将该路由从错误缓存记录中移除
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      // 在连接池中添加该连接
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      // 如果有其他复数连接到相同地址, 则删除重复连接
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```
其实看方法注释, 我们大概可以知道这里做的就是返回一个连接, 首先会从连接池中来, 如果连接池中没有对应连接, 则再重新新建一个连接.

具体的注释都在代码里了, 我们再看下其中几个调用方法.
首先我们将即将要释放的连接指向当前的连接, 通过调用`releaseIfNoNewStreams`方法, 返回需要关闭的socket
我们来看下`releaseIfNoNewStreams`方法的代码
``` java
/**
   * 如果当前连接无法新建流, 释放当前连接, 并且返回需要关闭的socket
   * 由于http2复数请求会使用同一个连接, 所以可能存在当前连接限制后续的请求
   */
  private Socket releaseIfNoNewStreams() {
    // 断言锁持有
    assert (Thread.holdsLock(connectionPool));
    RealConnection allocatedConnection = this.connection;
    // 当当前连接不为空, 而且没有新的流被创建
    if (allocatedConnection != null && allocatedConnection.noNewStreams) {
      return deallocate(false, false, true);
    }
    // 正常情况, 没有需要被关闭的socket返回
    return null;
  }
```
可以看到只有当当前连接存在, 而且不允许有新的流产生的时候, 才会返回执行`deallocate(false, false, true)`后的结果, 正常的情况下, 没有需要被关闭的socket返回
关于`deallocate`方法, 可以看下面的代码
``` java
private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
    assert (Thread.holdsLock(connectionPool));

    if (streamFinished) {
      this.codec = null;
    }
    if (released) {
      this.released = true;
    }
    Socket socket = null;
    if (connection != null) {
      if (noNewStreams) {
        connection.noNewStreams = true;
      }
      if (this.codec == null && (this.released || connection.noNewStreams)) {
        // 释放连接
        release(connection);
        // 如果这个连接的当前的流为空
        if (connection.allocations.isEmpty()) {
          // 当连接的流为0时候的记录时间戳
          connection.idleAtNanos = System.nanoTime();
          // 判断连接是否闲置
          if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
            // 如果闲置, 则返回需要关闭的socket
            socket = connection.socket();
          }
        }
        // 回收
        connection = null;
      }
    }
    return socket;
  }

  private void release(RealConnection connection) {
    for (int i = 0, size = connection.allocations.size(); i < size; i++) {
      Reference<StreamAllocation> reference = connection.allocations.get(i);
      if (reference.get() == this) {
        connection.allocations.remove(i);
        return;
      }
    }
    throw new IllegalStateException();
  }
```
可以看到这里具体做的就是, 编解码类对象`codec`赋值为null, 调用`release`释放连接, 当这个`connection`没有连接流的时候, 一并判断连接是否闲置, 如果闲置, 则返回对应的`socket`, 并将当前的`connection`赋值为null.而这里的`release`方法主要做的就是移除连接对应流引用的移除.

我们回头去看`findConnection`方法内下一步
``` java
if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        // 如果当前连接不为空, 就说明这个连接是可以用的
        result = this.connection;
        releasedConnection = null;
      }
```
我们知道前面调用`releaseIfNoNewStreams`的时候, 如果有返回socket, 那么connection也会被置为null, 所以这里connection不为空, 说明就是现在的连接是可以用的, 那么需要释放连接的对象就为null, 没必要被释放.
``` java
// 如果没有现成的连接
      if (result == null) {
        // Attempt to get a connection from the pool.
        // 尝试从连接池中获取
        Internal.instance.get(connectionPool, address, this, null);
        // 如果有复用的连接
        if (connection != null) {
          // 表示找到连接池可复用的连接
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
```
而如果没有可用的连接, 那么就会从连接池中尝试获取
``` java
  /**
   * 返回一个可重用的连接, 如果没有对应连接存在, 则返回null
   */
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    // 断言锁的持有
    assert (Thread.holdsLock(this));
    // 遍历
    for (RealConnection connection : connections) {
      // 判断连接是否可复用
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    // 添加一个流的引用
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```
可以看到, 如果当前连接池中有连接可复用, 则会将新的流引用添加在`connection.allocations`中.
``` java
// 关闭socket
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      // 如果有存在已分配的连接或者是连接池内可复用的连接, 则直接返回该连接对象
      return result;
    }
```
再回到主方法内, 不论是否有找到可用的连接, 都会关闭socket, 然后根据是否存在需要释放的连接, 回调`eventListener.connectionReleased`, 并根据是否找到连接池内可用连接, 回调`eventListener.connectionAcquired`.当有实际可用的连接的时候, 那么直接返回该连接对象.
``` java
boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      // 切换路由
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          // 获取可复用的连接
          Internal.instance.get(connectionPool, address, this, route);
          // 如果存在可复用连接
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }
      // 如果没有找到可复用的连接
      if (!foundPooledConnection) {
        // 如果当前路由为空
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        // 创建新的连接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }
    // If we found a pooled connection on the 2nd time around, we're done.
    // 如果第二次有找到, 则返回复用的连接
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }
```
当仍然没有找到连接的时候, 那么就会切换路由, 再次从连接池内找对应路由可复用的连接, 如果有找到, 则返回这次复用的连接对象.

``` java
// Do TCP + TLS handshakes. This is a blocking operation.
    // 做三次握手
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    // 将该路由从错误缓存记录中移除
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      // 在连接池中添加该连接
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      // 如果有其他复数连接到相同地址, 则删除重复连接
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
```
但如果这次仍然没有找到对应可用的连接, 则只好新建连接, 并将流引用加到对应的`conection`对象中, 然后做三次握手, 并将对应的路由从错误缓存中移除.
最后还做了重复连接的去重的工作, 然后再返回这个新建的连接对象.

截止至此, 寻找可用连接的代码分析就完成了.

回头再看下`HttpCodec的初始化`
``` java
public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```
主要就是根据判断是Http1还是Http2来判断新建哪个编解码类.
## 总结
可以看出, `ConnectionInterceptor`拦截器, 主要做的是,
1. 获取当前连接(`connection`), 如果不可用, 则从连接池中获取可复用连接, 如果仍然获取不到, 则新建连接
2. 通过连接, 生成`HttpCodec`对象

---
layout: article
title: okHttp3解析（四）：连接机制
aside:
  toc: true
key: Android
---

上一节介绍了 `CacheInterceptor` 缓存机制，如果符合缓存的条件则拦截器将拦截返回 response 。但如果缓存丢失，则传到下一个拦截器 `ConnectInterceptor` 进行处理。

在 `CacheInterceptor` 中只有一个 `intercept` 函数，源码如下：

```
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```


从上面代码的内容，可以观察到最开始通过 `realChain` 获取到 `streamAllocation` 对象。`StreamAllocation` 类是用于配置网络请求所需要的网络组件，对stream进行分配，例如 `HTTP/1.x` 连接一次只能传送1个流，`HTTP/2` 通常会传送多个流。

而 `StreamAllocation` 是在 `RetryAndFollowUpInterceptor.intercept` 中已经初始化。

```
StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
``` 

通过调用 `StreamAllocation.newStream` 方法可以获得 `HttpCodec` 对象，这里需要注意的是需要对 `GET` 方法进行额外的检查。这里的 `HttpCodec` 是一个接口，用于对 HTTP request 编码以及对 HTTP response 解码。

最后调用 `StreamAllocation.connection` 获取网络链路。

## StreamAllocation

以下是 `StreamAllocation` 的构造函数：

```
public StreamAllocation(ConnectionPool connectionPool, Address address, Call call, EventListener eventListener, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.call = call;
    this.eventListener = eventListener;
    this.routeSelector = new RouteSelector(address, routeDatabase(), call, eventListener);
    this.callStackTrace = callStackTrace;
}
```

可以观察到，一开始传入的 `connectionPool` 对象，是根据 `RetryAndFollowUpInterceptor` 中传入的 `client.connectionPool`。具体 `ConnectionPool` 类的作用，是用来管理 HTTP 和 HTTP/2 的连接复用以减少延迟，享用相同的地址的 HTTP request 可能享用相同的连接。该类实现了那些连接要保持打开提供给未来使用。

同时，`RouteSelector` 类是用于选择要连接到原本服务器的路由，每个连接都需要选择代理服务器，IP地址和TLS模式。 连接也可以被回收。

之后查看，在 `CacheInterceptor.intercept` 中被调用的 `newStream` 方法。

```
public HttpCodec newStream(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
            writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
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

前面的部分是通过 `chain` 获得 `connectTimeout` 连接超时，`readTimeout` 读超时， `writeTimeout` 写超时，`pingInterval` ping间隔，和 `retryOnConnectionFailure` 是否连接失败要重新连接。

```
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // 如果这是个全新的连接，可以跳过检查
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // 进行检查（可能会慢）以确认在连接池中的连接是否是好的，如果不是则从池中取出重新开始。
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

`findHealthyConnection` 是用于获取一个连接，判断是否可用，如果不可用就一直循环直到找到可用的连接。其中寻找连接的过程使用的是 `findConnection`，返回用于管理新的stream的连接。如果存在连接的话，则首选已有的连接，然后是连接池，最后是一个新连接。以下是该方法的源码：

```
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // 尝试去使用一个已经被分配的现有连接，我们需要去注意，因为现有的连接可能被限制去创建新的stream。
      releasedConnection = this.connection;
      // 如果保留的连接限制了新stream的创建，则释放连接。使用HTTP / 2，多个请求共享同一连接，因此有可能限制我们的连接在后续请求期间创建新的流。
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // 如果连接没有被释放，则将已经分配的连接作为结果。
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // 如果连接从未标记被获得，也不标记释放。
        releasedConnection = null;
      }

      if (result == null) {
        // 尝试从连接池中获取连接
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          // 如果找到了连接，则将连接池的中找到的连接作为结果。
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    // 关闭需要释放的连接
    closeQuietly(toClose);
    if (releasedConnection != null) {
      // 释放连接后调用
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      // 获取连接后调用
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // 如果找到了已经分配或者是连接池中的连接，则作为结果直接返回
      return result;
    }

    // 如果需要路由选择，则进行一次。这是一个阻塞过程。
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        List<Route> routes = routeSelection.getAll();
        // 现在我们有一组IP地址，尝试从连接池中获取一个连接。由于连接合并可能会找到匹配的。
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      // 如果没有在连接池中找到连接
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // 创建一个连接，并且立刻分配当前的stream流给它。这可以让异步cancel()去中断将要进行的挥手
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // 如果我们在第二次尝试从连接池中获取，则将其作为结果返回。
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // 进行 TCP + TLS 握手过程，这是一个阻塞过程。
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // 池连接
      Internal.instance.put(connectionPool, result);

      // 如果同时创建了到同一地址的另一个多路复用连接，则释放该连接并获取该连接。
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
```

经过上面的代码以及注释，我们可以将寻找连接的过程分为以下几步：
- 先获取一个已经存在的 `connection`，如果保留的连接被限制，则通过函数 `releaseIfNoNewStreams` 进行释放。
- 如果 `connection` 不为空且没有被释放，则使用获取到的已有的 `connection` 作为查找到的结果。
- 如果没有获取到，则尝试通过 `Internal.instance.get` 函数去获取连接池中的 `connection` ，如果获取到，则作为查找到的结果。没有获取到则将 `route` 作为要去选择的路由。
- 通过 `RouteSelection` 去得到路由列表，在路由的一组 IP 中，调用 `Internal.instance.get` 函数获取相匹配 `connection` 作为result。
- 没有获取的 `connection` 则创建新的连接，进行 TCP + TLS 握手过程，然后将该连接放入连接池中。
- 获取成功过后，释放之前创建的连接的相关资源。
- 获取不成功，则进行重新连接。

流程图：
!["findConnection流程图"]({{site.url}}/assets/images/android/okhttp/findConnection流程图.png "findConnection流程图")

`findHealthyConnection` 的最后部分，`noNewStreams` 方法用于防止在该分配的连接上创建新的stream。

## RealConnection

根据上面 `StreamAllocation.findConnection` 返回的是一个 `RealConnection` 对象，之后是对 `RealConnection` 类的分析。下面是 `RealConnection.connect` 方法的源码部分：

```
public void connect(int connectTimeout, int readTimeout, int writeTimeout,
    int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
    EventListener eventListener) {
  if (protocol != null) throw new IllegalStateException("already connected");

  RouteException routeException = null;
  List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
  ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

  if (route.address().sslSocketFactory() == null) {
    if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
      throw new RouteException(new UnknownServiceException(
          "CLEARTEXT communication not enabled for client"));
    }
    String host = route.address().url().host();
    if (!Platform.get().isCleartextTrafficPermitted(host)) {
      throw new RouteException(new UnknownServiceException(
          "CLEARTEXT communication to " + host + " not permitted by network security policy"));
    }
  } else {
    if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
      throw new RouteException(new UnknownServiceException(
          "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
    }
  }

  while (true) {
    try {
      if (route.requiresTunnel()) {
        connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
        if (rawSocket == null) {
          // We were unable to connect the tunnel but properly closed down our resources.
          break;
        }
      } else {
        connectSocket(connectTimeout, readTimeout, call, eventListener);
      }
      establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
      eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
      break;
    } catch (IOException e) {
      closeQuietly(socket);
      closeQuietly(rawSocket);
      socket = null;
      rawSocket = null;
      source = null;
      sink = null;
      handshake = null;
      protocol = null;
      http2Connection = null;

      eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

      if (routeException == null) {
        routeException = new RouteException(e);
      } else {
        routeException.addConnectException(e);
      }

      if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
        throw routeException;
      }
    }
  }

  if (route.requiresTunnel() && rawSocket == null) {
    ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
        + MAX_TUNNEL_ATTEMPTS);
    throw new RouteException(exception);
  }

  if (http2Connection != null) {
    synchronized (connectionPool) {
      allocationLimit = http2Connection.maxConcurrentStreams();
    }
  }
}
```


## Route



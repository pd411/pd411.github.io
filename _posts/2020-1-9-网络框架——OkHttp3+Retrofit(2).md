---
layout: article
title: okHttp3解析（二）：拦截器（责任链模式）
aside:
  toc: true
key: Android
---

接着上一部分的内容，在`AsyncCall`类中同步`execute`或异步`executeOn`两个方法中均有`getResponseWithInterceptorChain()`方法调用。以下是该方法的内容：
```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
}
```
该方法响应的获取核心部分，从上面的代码内容一开始先创建了一个`Interceptor`类（拦截器）列表`List<Interceptor> interceptors = new ArrayList<>()`。`RetryAndFollowUpInterceptor`、`BridgeInterceptor`、`CacheInterceptor`、`ConnectInterceptor`和`CallServerInterceptor`这五个类都为`Interceptor`的子类。
```
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;
  interface Chain {
    Request request();
    Response proceed(Request request) throws IOException;
    @Nullable Connection connection();
    Call call();
    int connectTimeoutMillis();
    Chain withConnectTimeout(int timeout, TimeUnit unit);
    int readTimeoutMillis();
    Chain withReadTimeout(int timeout, TimeUnit unit);
    int writeTimeoutMillis();
    Chain withWriteTimeout(int timeout, TimeUnit unit);
  }
}
```
在该类中，只有一个`intercept`方法和一个`Chain`接口。而这里用到了**责任链模式**，就是用来处理相关事务责任的一条执行链，执行链上有多个节点，每个节点都有机会（条件匹配）处理请求事务，如果某个节点处理完了就可以根据实际业务需求传递给下一个节点继续处理或者返回处理完毕。
在这里处理的就是通过`RealInterceptorChain`类将interceptor串连起来，再调用`Response response = chain.proceed(originalRequest)`。
```
  public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
      throws IOException {
    ...
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
	...
    return response;
  }
```
其余部分的内容是关于异常处理，上面部分的代码将下一层的chain传入`interceptor.intercept`方法，获取到response。通过查看对应的`Interceptor`子类，可以发现`intercept`的返回值Response都是通过`chain.proceed`方法获取。这里就类似于递归一层一层的传递下去，而传入的参数也就是下一层拦截器（Interceptor）的Chain。

这样能够得到的优势在于每一层的Interceptor可以去决定是否自己去处理Response或者是交给下一层的Chain去处理。如果上一节内容中的流程图。

![Interceptor流程图]({{site.url}}/assets/images/android/okhttp/2.jpg "Interceptor流程图")

> 关键点：责任链模式
**Interceptor**：每一层的拦截器接口，需要进行实现
**Chain**：串连拦截器的链

## RetryAndFollowUpInterceptor
`RetryAndFollowUpInterceptor`是Chain中一个拦截器，负责了HTTP请求的重定向。**重定向**就是通过各种方法将各种网络请求重新定个方向转到其它位置，分为网站调整（网页目录改变）；网页被移到一个新地址；网页拓展名改变（如应用需要把.php改成.Html或.shtml）。
```
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();
    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
	// 连接前的准备
      transmitter.prepareToConnect(request);
	  // 连接被取消抛出异常
      if (transmitter.isCanceled()) {
        throw new IOException("Canceled");
      }
      Response response;
      boolean success = false;
      try {
	  // 处理下一个chain
        response = realChain.proceed(request, transmitter, null);
        success = true;
      } catch (RouteException e) {
	  // 尝试与路由连接失败，request没有被送到
        if (!recover(e.getLastConnectException(), transmitter, false, request)) {
          throw e.getFirstConnectException();
        }
       // 满足重定向条件，重试
        continue;
      } catch (IOException e) {
	  // 尝试与服务器通信失败，request可能被送到
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, transmitter, requestSendStarted, request)) throw e;
       // 满足重定向条件，重试
        continue;
      } finally {
	  // 网络调用出的一些异常，释放一些资源
        if (!success) {
          transmitter.exchangeDoneDueToException();
        }
      }
	  // 如果存在前置的response，则去绑定它，让其body为空
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }
      Exchange exchange = Internal.instance.exchange(response);
      Route route = exchange != null ? exchange.connection().route() : null;
	  // 根据得到的response去获取重定向的request
      Request followUp = followUpRequest(response, route);
	  // 如果followUp为空则，表示不需要在重定向
      if (followUp == null) {
        if (exchange != null && exchange.isDuplex()) {
          transmitter.timeoutEarlyExit();
        }
        return response;
      }
      RequestBody followUpBody = followUp.body();
      if (followUpBody != null && followUpBody.isOneShot()) {
        return response;
      }
      closeQuietly(response.body());
      if (transmitter.hasExchange()) {
        exchange.detachWithViolence();
      }
	  // 如果重定向的次数超过20，则抛出异常
      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
	  // 修改下一次重定向的request
      request = followUp;
	  // 修改前置response为当前response
      priorResponse = response;
	  // 在进行重定向循环
    }
  }
```
通过以上的内容，将重定向的步骤分为：（1）先进行连接前的准备；（2）通过`realChain.proceed()`处理下一层的Chain，获得response，如果连接失败则抛出异常，重新执行循环里的内容；（3）根据所获得的response和route去获取下一次的request，如果不需要重定向则返回当前response；（4）修改下一次重定向的request以及priorResponse，重新执行循环。


## BridgeInterceptor
`BridgeInterceptor`用于从应用的代码到网络的代码的桥梁。首先它将用户的request转换为网络的request，然后进行网络出，最后它将网络的response转换为用户的response。下面是`BridgeInterceptor.intercept`方法的源码。
```
 @Override public Response intercept(Chain chain) throws IOException {
 // 获取上一层chain的request
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
	// 通过userRequest去设置一些头信息
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
  // 设置host
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
	// 设置connection
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // 添加Accept-Encoding: gzip
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }
	// 将userRequest中的cookies设置进头信息中
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }
	// 设置user-agent
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
	// 进行处理获取服务器repsonse
    Response networkResponse = chain.proceed(requestBuilder.build());
	// 根据服务端的响应构建新的Response，并将userRequest设置为其request
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
 // 若之前设置了gzip压缩且response中也包含了gzip压缩，则进行gzip解压
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
```
通过上面的代码，可以观察到在该Interceptor中，主要的作用是对request的header进行配置，例如`Content-Type`、`Content-Length`、`Host`和`Connection`等。将处理过后的request传入Chain获取到response。在Response阶段，进行gzip的解压。
以下有几点需要注意：
- 开发者没有添加`Accept-Encoding`时，自动添加`Accept-Encoding: gzip`
- 自动添加了`Accept-Encoding`，会对request，response进行自动解压
- 手动添加了`Accept-Encoding`，不负责进行解压
- 自动解压时移除`Content-Length`，所以上层Java代码想要contentLength时为-1
- 自动解压时移除`Content-Encoding`
- 自动解压时，如果是分块传输编码，`Transfer-Encoding: chunked`不受影响。

## CacheInterceptor
`CacheInterceptor`是服务器来自缓存的请求并将响应写入缓存。以下是`CacheInterceptor.intercept`方法源码：
```
  @Override public Response intercept(Chain chain) throws IOException {
  // 查看是否配置了缓存，并尝试从缓存中取一次
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
 // 初始化新的缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
	// 从缓存策略中获取新的网络request
    Request networkRequest = strategy.networkRequest;
	// 从缓存策略中获取到新的缓存response
    Response cacheResponse = strategy.cacheResponse;
	// 通过缓存策略设置缓存监测
    if (cache != null) {
      cache.trackResponse(strategy);
    }
	// 如果读取出得缓存不能应用，则关闭它
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); 
	  }

    // 如果网络被禁止，同时缓存不可用，则直接返回一个构建失败的response
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
	// 如果不需要网络请求，则直接返回
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
	// 如果缓存无效，同时需要网络请求，则传入下一层chain取，获取response
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
	// 如果IO出现了冲突，则回收资源
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
    // 如果存在缓存，且网络请求返回为304（not modified），则对通过cacheResponse对缓存进行更新
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
	// 获取网络response
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
	// 对响应进行缓存
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
        }
      }
    }
    return response;
  }
```
根据上面的代码总结出一下步骤：
- 先获取是否配置了缓存，并且去获取response作为`cacheCandidate`
- 通过当前的时间、上一层的chain以及`cacheCandidate`读取缓存策略`CacheStrategy`
- 从缓存策略中获取到网络请求`networkRequest`和缓存响应`cacheResponse`，并且设置缓存监测
- 如果读取出得`cacheCandidate`不能应用（`cacheCandidate != null && cacheResponse == null`），则关闭它
- 如果网络和缓存都不可用（`networkRequest == null && CacheResponse == null`），则直接返回连接失败（504）的请求
- 如果网络不可用（`networkRequest == null`）且有缓存，则直接返回`CacheResponse`
- 网络可用的情况，则传到下一层`chain.proceed(networkRequest)`，获取`networkResponse `
- 如果存在缓存（`cacheResponse != null`），且网络请求返回为304（not modified），则对通过`cacheResponse`对缓存进行更新
- 通过`networkResponse `去构建`response`

## ConnectInterceptor
`ConnectInterceptor`是用于打开和目标服务器的连接，并且处理下一个Interceptor。下面是`ConnectInterceptor.intercept`的源码：
```
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();
    // 我们需要网络来满足此要求。 可能用于验证条件GET。
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    return realChain.proceed(request, transmitter, exchange);
  }
```
上面的代码，具体是通过之前的`realChain`获取到`transmitter`，之后调用`transmitter.newExchange`方法获取到`Exchange`。
`Exchange`是一个单独的HTTP请求和相应的对。是将连接管理和事件监听进行分层，事件监听在`ExchangeCodec`，它是处理实际的`IO`。

## CallServerInterceptor
`CallServerInterceptor`是最后一个拦截器，这个类是对服务器的数据进行交换、读取。下面是`CallServerInterceptor.intercept`方法的源码：
```
  @Override public Response intercept(Chain chain) throws IOException {
  // 根据上一层的chain获取request和exchange
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Exchange exchange = realChain.exchange();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();
	// 将请求头写入
    exchange.writeRequestHeaders(request);

    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
	// 判断request的实体是否为空
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
	  // 在发送request实体之前，如果有一个"Expect: 100-continue"头在request里面，等待一个有"HTTP/1.1 100 Continue"的response头。相当于一次握手，表示与客户端和服务器开始建立连接可以发送消息
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
	  // 将请求头刷新入request中
        exchange.flushRequest();
        responseHeadersStarted = true;
		// 在获取response头之前的操作
        exchange.responseHeadersStart();
		// 获取response头存入responseBuilder中
        responseBuilder = exchange.readResponseHeaders(true);
      }
      if (responseBuilder == null) {
	  // 判断request的实体是否是双工
        if (request.body().isDuplex()) {
		// 准备一个双工的body以至于应用之后能发送一个request实体
          exchange.flushRequest();
		  // 创建一个request实体流
          BufferedSink bufferedRequestBody = Okio.buffer(
              exchange.createRequestBody(request, true));
          request.body().writeTo(bufferedRequestBody);
        } else {
		  // 如果满足"Expect: 100-continue"，则写入request实体
          BufferedSink bufferedRequestBody = Okio.buffer(
              exchange.createRequestBody(request, false));
          request.body().writeTo(bufferedRequestBody);
          bufferedRequestBody.close();
        }
      } else {
        exchange.noRequestBody();
        if (!exchange.connection().isMultiplexed()) {
		  // 如果满足 "Expect: 100-continue"，则组织HTTP/1的连接，避免重复使用。否则，我们仍然有义务传输请求主体，以使连接保持一致状态。
          exchange.noNewExchangesOnConnection();
        }
      }
    } else {
      exchange.noRequestBody();
    }
    if (request.body() == null || !request.body().isDuplex()) {
      exchange.finishRequest();
    }
    if (!responseHeadersStarted) {
      exchange.responseHeadersStart();
    }
    if (responseBuilder == null) {
	// 获取response头
      responseBuilder = exchange.readResponseHeaders(false);
    }
	// 通过responseBuilder获得response的内容
    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
	// 得到响应码
    int code = response.code();
    if (code == 100) {
      // 如果响应码是100的话，在获取一次response
      response = exchange.readResponseHeaders(false)
          .request(request)
          .handshake(exchange.connection().handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();

      code = response.code();
    }
    exchange.responseHeadersEnd(response);
    if (forWebSocket && code == 101) {
      // 如果响应码为101，则设置一个空的实体
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
	// 获取response的实体信息
      response = response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build();
    }
	// 如果设置了连接关闭，则断开
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      exchange.noNewExchangesOnConnection();
    }
	    //如果状态码是204或者205，则抛出协议异常。
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
    return response;
  }
```
根据上面代码进行总结：
- 写入**request header**，`exchange.writeRequestHeaders(request)`
- 通过流的方式，写入**request body**，
```
BufferedSink bufferedRequestBody = Okio.buffer(
              exchange.createRequestBody(request, true));
          request.body().writeTo(bufferedRequestBody);
```
- 读取**response header**，`responseBuilder = exchange.readResponseHeaders(false)`
- 读取**response body**，
```
response = response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build();
```
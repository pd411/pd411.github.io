---
layout: article
title: okHttp3解析（三）：缓存机制
aside:
  toc: true
key: Android
---

通过上一节，拦截器中`CacheInterceptor`的内容，这一节的内容是关于OkHttp的缓存机制。从`intercept`部分开始进行分析：
```
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
```
这是`intercept`开始部分，`cache`就是`OkHttpClient.cache(cache)`配置的对象，而该对象是来自`InternalCache`。

## InternalCache
`InternalCache`是一个接口，官方注解上说OkHttp的内部缓存接口，应用不能去实现该接口，使用`okhttp3.Cache`去替代。
```
public interface InternalCache {
  @Nullable Response get(Request request) throws IOException;
  @Nullable CacheRequest put(Response response) throws IOException;
  void remove(Request request) throws IOException;te(Response cached, Response network);
  void trackConditionalCacheHit();
  void trackResponse(CacheStrategy cacheStrategy);
}
```
通过上面的部分，可以观察到这个接口里面包括了一些对Cache的获取、增加、删除、更新等功能。

## Cache
而该类是在`RealCall.getResponseWithInterceptorChain`方法中`interceptors.add(new CacheInterceptor(client.internalCache()))`，可以查询到`Cache cache`。通过`OkHttpClient.cache`进行设置：
```
    public Builder cache(@Nullable Cache cache) {
      this.cache = cache;
      this.internalCache = null;
      return this;
    }
```
在`Cache`类中，可以看到该类是由**DiskLruCache**方法进行实现。

```
  final InternalCache internalCache = new InternalCache() {
    @Override public @Nullable Response get(Request request) throws IOException {
      return Cache.this.get(request);
    }
    @Override public @Nullable CacheRequest put(Response response) throws IOException {
      return Cache.this.put(response);
    }
    @Override public void remove(Request request) throws IOException {
      Cache.this.remove(request);
    }
    @Override public void update(Response cached, Response network) {
      Cache.this.update(cached, network);
    }
    @Override public void trackConditionalCacheHit() {
      Cache.this.trackConditionalCacheHit();
    }
    @Override public void trackResponse(CacheStrategy cacheStrategy) {
      Cache.this.trackResponse(cacheStrategy);
    }
  };
```
同时`Cache`类通过初始化`InternalCache`一个对象进行实现，并调用自己方法实现。

接下来分析`Cache`类中`get`、`put`、`remove`、`update`、`trackConditionalCacheHit`和`trackResponse`方法。

### Cache.get
```
  @Nullable Response get(Request request) {
    // 将url通过MD5和HEX生成key
    String key = key(request.url());
	// 声明DiskLruCache的snapshot
    DiskLruCache.Snapshot snapshot;
	// 声明Entry类
    Entry entry;
    try {
	  // 读取url的缓存，存入snapshot中
      snapshot = cache.get(key);
	  // 如果snapshot为空，则缓存不存在，返回空
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
	 // 缓存不可读，则返回空
      return null;
    }
    try {
	// 通过snapshot获取实体
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }
	// 通过传入snapshot获取response实例
    Response response = entry.response(snapshot);
	// 判读是否匹配要求，否则关闭
    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }
    return response;
  }
```

从上面的内容，最开始的`key`方法，打开可以发现
```
  public static String key(HttpUrl url) {
    return ByteString.encodeUtf8(url.toString()).md5().hex();
  }
```
通过`md5`和`hex`方法将url进行转换。这里为什么要先对url转换成key，主要是因为DiskLruCache内部由**LinkedHashMap**进行维护。

方法中使用`Cache`中的`Entry`类对cache进行获取，在该类里面，通过`Okio`类进行数据的读取，并缓存入`Entry`中。

```
public Response response(DiskLruCache.Snapshot snapshot) {
  String contentType = responseHeaders.get("Content-Type");
  String contentLength = responseHeaders.get("Content-Length");
  Request cacheRequest = new Request.Builder()
      .url(url)
      .method(requestMethod, null)
      .headers(varyHeaders)
      .build();
  return new Response.Builder()
      .request(cacheRequest)
      .protocol(protocol)
      .code(code)
      .message(message)
      .headers(responseHeaders)
      .body(new CacheResponseBody(snapshot, contentType, contentLength))
      .handshake(handshake)
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(receivedResponseMillis)
      .build();
}
```
最后通过`Entry`的`response`方法获取到`Response`对象，上面是源码部分。上面的内容则是将缓存的内容存入`Response.Builder`中。

### Cache.put
```
@Nullable CacheRequest put(Response response) {
  // 先获取到request方法
  String requestMethod = response.request().method();
  // 对request的方法进行验证
  if (HttpMethod.invalidatesCache(response.request().method())) {
    try {
      // 如果request的方法为POST、PUT、PATCH、DELETE、MOVE，则删除现有的缓存
      remove(response.request());
    } catch (IOException ignored) {
      // The cache cannot be written.
    }
    return null;
  }
  if (!requestMethod.equals("GET")) {
    // 不缓存非GET的方法，虽然技术上可以运行去缓存HEAD请求和一些POST请求，但复杂度高，收益低
    return null;
  }
  // 如果header信息里面包含Vary，则不进行缓存
  if (HttpHeaders.hasVaryAll(response)) {
    return null;
  }
  // 通过response获取entry
  Entry entry = new Entry(response);
  DiskLruCache.Editor editor = null;
  try {
    // 通过url获取到editor
    editor = cache.edit(key(response.request().url()));
    if (editor == null) {
      return null;
    }
    // 将editor写入entry中
    entry.writeTo(editor);
    return new CacheRequestImpl(editor);
  } catch (IOException e) {
    abortQuietly(editor);
    return null;
  }
}
```

从开头的`HttpMethod.invalidatesCache`方法的源码：
```
  public static boolean invalidatesCache(String method) {
    return method.equals("POST")
        || method.equals("PATCH")
        || method.equals("PUT")
        || method.equals("DELETE")
        || method.equals("MOVE");     // WebDAV
  }
```
`POST`、`PATCH`、`PUT`、`DELETE`和`MOVE`这五个request method，都不进行缓存，调用`remove`方法。

在`put`中的重点在通过`response`去获取`Entry`，在根据url转换为key存入`Entry`对象中。

### Cache.remove
```
void remove(Request request) throws IOException {
  cache.remove(key(request.url()));
}
```
可以发现上面的`remove`的内容很简单，是调用了`cache`对象（DiskLruCache）的`remove`方法。而我们知道DiskLruCache是由一个`LinkedHashMap<String, Entry>`进行维护，key是根据url转成的key，而value是`DiskLruCache`中的`Entry`。

### Cache.update
```
void update(Response cached, Response network) {
  // 通过网络的response获取到entry
  Entry entry = new Entry(network);
  // 获取到cache的snapshot
  DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
  DiskLruCache.Editor editor = null;
  try {
    editor = snapshot.edit(); // Returns null if snapshot is not current.
    // 写入entry中，用network去替换cache
    if (editor != null) {
      entry.writeTo(editor);
      editor.commit();
    }
  } catch (IOException e) {
    abortQuietly(editor);
  }
}
```

在`update`方法中，则是使用`network`的response去替换`cache`中的response。

## CacheStrategy
该类用于通过一个request和缓存的response，去确定是否使用网络或者缓存。选择一个`CacheStrategy`可能会向request添加条件（如`GET`的` If-Modified-Since`），或者向缓存的response添加警告（如果缓存的数据可能已过时）。

```
public final @Nullable Request networkRequest;
public final @Nullable Response cacheResponse;
```

在`CacheStrategy`中，首先是两个`Request`对象：1. `networkRequest`表示在网络上发送的请求；如果此调用不使用网络，则返回null；2. `cacheResponse`表示在缓存的响应返回或验证； 如果此调用不使用缓存，则返回null。

接着里面有一个`isCacheable`方法，如果能储存一个之后还会被request的response，则返回true。
```
  public static boolean isCacheable(Response response, Request request) {
    // Always go to network for uncacheable response codes (RFC 7231 section 6.1),
    // This implementation doesn't support caching partial content.
    switch (response.code()) {
      case HTTP_OK:
      case HTTP_NOT_AUTHORITATIVE:
      case HTTP_NO_CONTENT:
      case HTTP_MULT_CHOICE:
      case HTTP_MOVED_PERM:
      case HTTP_NOT_FOUND:
      case HTTP_BAD_METHOD:
      case HTTP_GONE:
      case HTTP_REQ_TOO_LONG:
      case HTTP_NOT_IMPLEMENTED:
      case StatusLine.HTTP_PERM_REDIRECT:
        // These codes can be cached unless headers forbid it.
        break;

      case HTTP_MOVED_TEMP:
      case StatusLine.HTTP_TEMP_REDIRECT:
        // These codes can only be cached with the right response headers.
        // http://tools.ietf.org/html/rfc7234#section-3
        // s-maxage is not checked because OkHttp is a private cache that should ignore s-maxage.
        if (response.header("Expires") != null
            || response.cacheControl().maxAgeSeconds() != -1
            || response.cacheControl().isPublic()
            || response.cacheControl().isPrivate()) {
          break;
        }
        // Fall-through.

      default:
        // All other codes cannot be cached.
        return false;
    }

    // A 'no-store' directive on request or response prevents the response from being cached.
    return !response.cacheControl().noStore() && !request.cacheControl().noStore();
  }
```

在`CacheStrategy`中，维护了一个`Factory`类。下面是`Factory`的构造函数：
```
public Factory(long nowMillis, Request request, Response cacheResponse) {
  this.nowMillis = nowMillis;
  this.request = request;
  this.cacheResponse = cacheResponse;

  if (cacheResponse != null) {
    this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
    this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
    Headers headers = cacheResponse.headers();
    for (int i = 0, size = headers.size(); i < size; i++) {
      String fieldName = headers.name(i);
      String value = headers.value(i);
      if ("Date".equalsIgnoreCase(fieldName)) {
        servedDate = HttpDate.parse(value);
        servedDateString = value;
      } else if ("Expires".equalsIgnoreCase(fieldName)) {
        expires = HttpDate.parse(value);
      } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
        lastModified = HttpDate.parse(value);
        lastModifiedString = value;
      } else if ("ETag".equalsIgnoreCase(fieldName)) {
        etag = value;
      } else if ("Age".equalsIgnoreCase(fieldName)) {
        ageSeconds = HttpHeaders.parseSeconds(value, -1);
      }
    }
  }
}
```

通过`get`方法获取到`CacheStrategy`对象：
```
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();
  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // 禁止使用网络，并且缓存不足
    return new CacheStrategy(null, null);
  }
  return candidate;
}
```

该方法通过调用`getCandidate`去获取到`CacheStrategy`对象，下面是`getCandidate`的源码：
```
private CacheStrategy getCandidate() {
  // 如果没有缓存的response，则直接返回
  if (cacheResponse == null) {
    return new CacheStrategy(request, null);
  }

  // 如果缺少必要的TLS握手，则删除缓存的response
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // 使用isCacheable方法，监测response是否可被缓存
  if (!isCacheable(cacheResponse, request)) {
    return new CacheStrategy(request, null);
  }

  // 如果请求的Cache-Control中指定了no-cache，则不使用缓存的response
  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  CacheControl responseCaching = cacheResponse.cacheControl();
  // 计算当前缓存的response的存活时间以及缓存应当被刷新的时间
  long ageMillis = cacheResponseAge();
  long freshMillis = computeFreshnessLifetime();
  if (requestCaching.maxAgeSeconds() != -1) {
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }
  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }
  long maxStaleMillis = 0;
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }
  // 对未超过时限的缓存，直接采用缓存数据策略
  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    return new CacheStrategy(null, builder.build());
  }

  // 查找“If-None-Match”，“If-Modified-Since”去添加入header中，如果这两个信息都不存在，则采用网络的request
  String conditionName;
  String conditionValue;
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    return new CacheStrategy(request, null); 
  }
  // 若存在上述Header，则在原request中添加对应header,之后结合本地cacheResponse创建缓存策略
  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);
  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

综合上面代码，`CacheStrategy`中对于使用网络请求和缓存请求的总结，如下：
- `request != null && response == null`：采用网络request，不使用缓存request；
- `request == null && response != null`：采用缓存request，忽略网络request；
- `request != null and response != null`： 存在`Last-Modified`、`Etag`等相关数据，结合request及缓存中的response；
- `request == null and response == null`：不允许使用网络请求，且没有缓存，在`CacheInterceptor`中会构建一个`504`的response。




---
layout: article
title: okHttp3解析（一）：异步与同步
aside:
  toc: true
key: Android
---

## 使用方法：
1.添加依赖：
```
    implementation 'com.squareup.okhttp3:okhttp:4.3.0'
    testImplementation 'com.squareup.okhttp3:mockwebserver:4.3.0'
```

2.添加网络权限在AndroidManifest.xml中：
```
  <uses-permission android:name="android.permission.INTERNET" />
```

3.1对url进行Get请求：
```
    OkHttpClient client = new OkHttpClient();

    String run(String url) throws IOException {
        Request request = new Request.Builder()
                .url(url)
                .build();

        try (Response response = client.newCall(request).execute()) {
            return response.body().string();
        }
    }
```

3.2对服务器进行post请求
```
    public static final MediaType JSON
            = MediaType.get("application/json; charset=utf-8");

    OkHttpClient client = new OkHttpClient();

    String post(String url, String json) throws IOException {
        RequestBody body = RequestBody.create(json, JSON);
        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .build();
        try (Response response = client.newCall(request).execute()) {
            return response.body().string();
        }
    }
```

上面的get与post请求都按照一下步骤:
- 创建OkHttpClient对象
- 创建Request对象
- OkHttpClient对象调用newCall传入参数Request，调用**execute**方法（**同步**请求）

同时我们可以修改上方的代码改为enqueue方法进行**异步**请求：
```
client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(@NotNull okhttp3.Call call, @NotNull IOException e) {
                ...
            }

            @Override
            public void onResponse(@NotNull okhttp3.Call call, @NotNull Response response) throws IOException {
				...
            }
        });
```

## 详细分析：
### OkHttp3优点：
- 支持Http1.0、1.1和2.0，支持连接同一地址共享同一个socket
- 支持透明的Gzip压缩
- 通过连接池来减小响应延迟
- 请求缓存，避免重复请求
- 请求失败时自动重定向

### 源码分析：
通过上面的代码，先介绍OkHttpClient、Request、Response和Call几个类：
- OkHttpClient：
```
public OkHttpClient() {
	this(new Builder());
}
OkHttpClient(Builder builder) {
	this.dispatcher = builder.dispatcher;
	this.proxy = builder.proxy;
	this.protocols = builder.protocols;
	this.connectionSpecs = builder.connectionSpecs;
	...
}
```
 从上面代码内容分析，OkHttpClient有两种创建方法：一种是上面举例中的`OkHttpClient client = new OkHttpClient();`；另一种方法是：对象的构造函数通过传入builder对象进行初始化，也就是`OkHttpClient client = new OkHttpClient.Builder().build()`。
第一种通过构建Builder对象，下面是Builder类的构造方法：
```
public Builder() {
	dispatcher = new Dispatcher();
	protocols = DEFAULT_PROTOCOLS;
	connectionSpecs = DEFAULT_CONNECTION_SPECS;
	eventListenerFactory = EventListener.factory(EventListener.NONE);
	proxySelector = ProxySelector.getDefault();
	if (proxySelector == null) {
		proxySelector = new NullProxySelector();
	}
	cookieJar = CookieJar.NO_COOKIES;
	socketFactory = SocketFactory.getDefault();
	hostnameVerifier = OkHostnameVerifier.INSTANCE;
	certificatePinner = CertificatePinner.DEFAULT;
	proxyAuthenticator = Authenticator.NONE;
	authenticator = Authenticator.NONE;
	connectionPool = new ConnectionPool();
	dns = Dns.SYSTEM;
	followSslRedirects = true;
	followRedirects = true;
	retryOnConnectionFailure = true;
	callTimeout = 0;
	connectTimeout = 10_000;
	readTimeout = 10_000;
	writeTimeout = 10_000;
	pingInterval = 0;
}
```
 分析构造函数中的参数：
 - dispatcher - 调度器；
 - protocols - 默认支持的Http协议；
 - connectionSpecs - OkHttp连接配置；
 - eventListenerFactory - Call的状态监听器；
 - cookieJar - 默认没有Cookie；
 - socketFactory - socket的构造工厂；
 - hostnameVerifier、certificatePinner、proxyAuthenticator、authenticator - 安全相关的设置；
 - connectionPool - 连接池；
 - dns - DNS域名解析器。

 第二种是可以使用建造者模式单独去设定这些属性，默认情况下和第一种的参数是一样的。

 > 注：创建OkHttpClient应使用单例模式，因为每一次初始化OkHttpClient会新建它自己的连接池和线程池。

 通过上面的源码，可以观察到OkHttpClient运用到了Builder建造者模式，建造者模式使用多个简单的对象一步步构建成一个复杂的对象。

- Request：
 在Request类中，有一个Builder的内部类，其构造函数如下：
```
 public Builder() {
	 this.method = "GET";
	 this.headers = new Headers.Builder();
 }
```
初始化默认方法HTTP请求方法为get，同时初始化一个Http请求头文件。接下里，观察Builder类中的函数，首先是三种设定url的方法，都先判断传入的url是否为空：
```
public Builder url(HttpUrl url) {
	if (url == null) throw new NullPointerException("url == null");
	this.url = url;
	return this;
}
public Builder url(String url) {
	if (url == null) throw new NullPointerException("url == null");
	if (url.regionMatches(true, 0, "ws:", 0, 3)) {
		url = "http:" + url.substring(3);
	} else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
		url = "https:" + url.substring(4);
	}
	return url(HttpUrl.get(url));
}
public Builder url(URL url) {
	if (url == null) throw new NullPointerException("url == null");
	return url(HttpUrl.get(url.toString()));
}
```
接下来是一些Http的请求方式，基本的一些get、head、post、delete、put、patch方法：
```
public Builder get() {
	return method("GET", null);
}
public Builder head() {
	return method("HEAD", null);
}
public Builder post(RequestBody body) {
	return method("POST", body);
}
public Builder delete(@Nullable RequestBody body) {
	return method("DELETE", body);
}
public Builder delete() {
	return delete(Util.EMPTY_REQUEST);
}
public Builder put(RequestBody body) {
	return method("PUT", body);
}
public Builder patch(RequestBody body) {
	return method("PATCH", body);
}
public Builder method(String method, @Nullable RequestBody body) {
	if (method == null) throw new NullPointerException("method == null");
	if (method.length() == 0) throw new IllegalArgumentException("method.length() == 0");
	if (body != null && !HttpMethod.permitsRequestBody(method)) {
		throw new IllegalArgumentException("method " + method + " must not have a request body.");
	}
	if (body == null && HttpMethod.requiresRequestBody(method)) {
		throw new IllegalArgumentException("method " + method + " must have a request body.");
	}
	this.method = method;
	this.body = body;
	return this;
}
```
最后builder方法，创建Request对象，判断url是否为空：
```
public Request build() {
	if (url == null) throw new IllegalStateException("url == null");
	return new Request(this);
}
```

- Response：

- Call：这是一个接口，继承Clonable类，一个回调（call）是一个已经准备好被执行的请求，call可以被取消。因为这个对象代表单次的request/response对，**所以它不能被执行两次**。
在上面的代码中，对于`OkHttpClient`的`newCall`方法的调用，以下是`newCall`方法里的内容：
```
@Override public Call newCall(Request request) {
	return RealCall.newRealCall(this, request, false /* for web socket */);
}
```
可以注意到方法中`RealCall`，该类是一个接口Call的实现类，通过该类去完成`execute`、`enqueue`这两个**同步**以及**异步**的请求方式。
```
@Override public Response execute() throws IOException {
   synchronized (this) {
   if (executed) throw new IllegalStateException("Already Executed");
   	executed = true;
   }
   transmitter.timeoutEnter();
   transmitter.callStart();
   try {
	   client.dispatcher().executed(this);
	   return getResponseWithInterceptorChain();
   } finally {
   	client.dispatcher().finished(this);
   }
}
@Override public void enqueue(Callback responseCallback) {
	synchronized (this) {
		if (executed) throw new IllegalStateException("Already Executed");
		executed = true;
	}
		transmitter.callStart();
		client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
在`execute`和`enqueue`方法中用到了`synchronized`， 去保证当前的执行只有一个，如果已经在执行了就抛出异常。

### 异步
![OkHttp的流程图]({{site.url}}/assets/images/android/okhttp/1.jpg "Okhttp流程图")

> 注：OkHttp中同步和异步的区别在于：同步直接执行`execute`方法；而异步需要先执行`enqueue`方法，通过`Dispatcher`类创建`executorService`线程池，进行`execute`方法。


这里引入一个关键的点**dispatcher**，也就是OkHttpClient构造函数里面介绍的调度器。先观察异步的`enqueue`方法，在该方法中`client.dispatcher().enqueue`传入的参数是一个`AsyncCall`对象，而该对象继承`NamedRunnable`类，该类是一个抽象类实现`Runnable`，重构`run`方法，在`run`中调用`execute`方法，通过子类去实现`execute`方法。接下里的内容是`Dispatcher`类：
```
private @Nullable ExecutorService executorService;
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```
首先有有一个**线程池**`executorService`以及三种类型的回调Deque进行缓存，分别是`readyAsyncCalls`（在异步回调队列中，将要被run）、`runningAsyncCalls`（正在运行的异步调用，包括没有完成却被取消的回调）、`runningSyncCalls`（正在运行的同步调用，包括没有完成却被取消的回调）。
```
void enqueue(AsyncCall call) {
	synchronized (this) {
	  readyAsyncCalls.add(call);
	  if (!call.get().forWebSocket) {
		AsyncCall existingCall = findExistingCallWithHost(call.host());
		if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
	  }
	}
	promoteAndExecute();
}
```
在`enqueue`方法中，先使用`synchronized`加锁将现在的call加入到`readyAsyncCalls`队列中，在判断是否当前call的host已经在队列中了，如果已经在队列中则复用。接下来调用`promoteAndExecute`方法，该方法将符合条件的回调从`readyAsyncCalls`传到`runningAsyncCalls`，并且运行它们在执行服务上。不能使用同步调用的方式，因为在执行的calls能调用用户代码。
 ```
 private boolean promoteAndExecute() {
	 assert (!Thread.holdsLock(this));

	 List<AsyncCall> executableCalls = new ArrayList<>();
	 boolean isRunning;
	 synchronized (this) {
	  for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
		AsyncCall asyncCall = i.next();
		if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
		if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.
		i.remove();
		asyncCall.callsPerHost().incrementAndGet();
		executableCalls.add(asyncCall);
		runningAsyncCalls.add(asyncCall);
	  }
	  isRunning = runningCallsCount() > 0;
	 }
	 for (int i = 0, size = executableCalls.size(); i < size; i++) {
	  AsyncCall asyncCall = executableCalls.get(i);
	  asyncCall.executeOn(executorService());
	 }
	 return isRunning;
 }
```
上面的内容是将`readyAsyncCalls`中的内容移到`executableCalls`和`runningAsyncCalls`中，接着对`executableCalls`中的call进行执行。如果`runningAsyncCalls`为空则返回false，如果不为空则返回true。
```
void executeOn(ExecutorService executorService) {
  assert (!Thread.holdsLock(client.dispatcher()));
  boolean success = false;
  try {
    executorService.execute(this);
    success = true;
  } catch (RejectedExecutionException e) {
    InterruptedIOException ioException = new InterruptedIOException("executor rejected");
    ioException.initCause(e);
    transmitter.noMoreExchanges(ioException);
    responseCallback.onFailure(RealCall.this, ioException);
  } finally {
    if (!success) {
      client.dispatcher().finished(this); // This call is no longer running!
    }
  }
}
```
而AsyncCall中的`executeOn`方法主要的作用是使用`executorService`（线程池）去执行当前异步call，如果失败则清空当前的异步call。AsyncCall类中对`execute`进行了实现：
```
@Override protected void execute() {
	  boolean signalledCallback = false;
	  transmitter.timeoutEnter();
	  try {
		Response response = getResponseWithInterceptorChain();
		signalledCallback = true;
		responseCallback.onResponse(RealCall.this, response);
	  } catch (IOException e) {
		if (signalledCallback) {
		  // Do not signal the callback twice!
		  Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
		} else {
		  responseCallback.onFailure(RealCall.this, e);
		}
	  } catch (Throwable t) {
		cancel();
		if (!signalledCallback) {
		  IOException canceledException = new IOException("canceled due to " + t);
		  canceledException.addSuppressed(t);
		  responseCallback.onFailure(RealCall.this, canceledException);
		}
		throw t;
	  } finally {
		client.dispatcher().finished(this);
	  }
}
```
首先` transmitter.timeoutEnter()`是timeout的计时，如果成功则会调用`getResponseWithInterceptorChain()`返回`Response`之后调用`Callback.onResponse`通知请求成功；如果请求失败则`Callback.onFailure`通知请求失败。最后无论请求成功或者失败都会调用`client.dispatcher().finished`。

以上的内容就是okHttp3里面的异步处理，总结：（1）dispatcher中线程池调度实现了高并发，低阻塞；（2）采用Deque作为缓存，先进先出的顺序执行；（3）任务在try/finally中调用了finished函数，控制任务队列的执行顺序，而不是采用锁，减少了编码复杂性提高性能。

### 同步
而同步直接调用`RealCall`类中的`execute`方法：
```
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      client.dispatcher().executed(this);
      return getResponseWithInterceptorChain();
    } finally {
      client.dispatcher().finished(this);
    }
  }
```
该方法直接调用了`client.dispatcher().executed`方法
```
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```
跳转到`Dispatcher`类的`executed`方法，直接将Call加入到`runningSyncCalls`队列中。

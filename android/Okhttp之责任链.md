# Okhttp之责任链

Okhttp简介

基本用法

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

这里大概分为四步

1. 构建OkHttpClient对象,配置公共的网络请求设置,如超时时间,拦截器等
2. 构建Request对象,主要有url,method,RequestBody等
3. 生成call对象
4. 获得Response对象

OkhttpClient,Request的构建都使用了建造者模式

接下来通过这个例子分析整个流程:

```java
#OkhttpClient.java
/**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

```

OkhttpClient的newCall方法其实是调用``RealCall`` 的一个静态方法,跟进去看一下

```java
#RealCall.java

final class RealCall implements Call {

.....
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

}
```

这里逻辑很简单,就是new了一个``RealCall``的实例,而``RealCall`` 则是``Call`` 接口的一个实现类,接下里看一下Call接口都定义了哪些方法:

```java
/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this   object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call extends Cloneable {
  /** Returns the original request that initiated this call. */
  Request request();
  
  Response execute() throws IOException;
    
  void enqueue(Callback responseCallback);

  /** Cancels the request, if possible. Requests that are already complete cannot be canceled. */
  void cancel();

  /**
   * Returns true if this call has been either {@linkplain #execute() executed} or {@linkplain
   * #enqueue(Callback) enqueued}. It is an error to execute a call more than once.
   */
  boolean isExecuted();

  boolean isCanceled();

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```

对于Call接口的定义,官方给出了解释:

> A call is a request that has been prepared for execution. A call can be canceled. As this   object
>
> represents a single request/response pair (stream), it cannot be executed twice.

翻译: 一个Call是为执行而准备的请求,它能够被取消,这个对象代表一个请求/响应对(流),不能够被执行两次.

大致就是Call相当一个真正用于网络请求的任务,它可以被取消,它里面的请求和响应是一一对应的,不能够被执行两次,大概可能导致状态异常吧(瞎猜的),所以每个请求需要新建一个Call的实例,不能够复用之前的.

Call中也包含了关键的用于同步请求的execute()方法,用于异步请求的enqueue()方法.

接下来看一下``RealCall``对于``execute()``的实现:

```java
#RealCall.java

@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

首先判断executed是否为true,如果是true则抛出异常,同时也保证了Call只能够被执行一次.

首先分析``client.dispatcher().executed(this);`` 

```java
#Dispatcher.java

public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
    
    
  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
    
}

```

可以看到``Dispatcher`` 其实就是一个**任务调度器** ,它里面维护了线程池,和三个任务队列(双向队列)

- ``readyAsyncCalls`` : 等待异步请求的队列
- ``runningAsyncCalls``: 运行的异步请求队列
- ``runningSyncCalls`` : 运行的同步请求队列

executed()方法则是将当前同步请求任务加入到同步请求队列中

这里先提一下``Interceptor`` ,也就是我们所说的拦截器.那它到底是啥呢?

    #Interceptor.java
    
    public interface Interceptor {
    
      Response intercept(Chain chain) throws IOException;
    
      interface Chain {
    
        Request request();
    
    Response proceed(Request request) throws IOException;
    
    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
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
 没错,它就是一个接口,而且只有一个intercept方法,它接收一个实现了``Chain`` 接口的对象,并返回一个Response对象.

接下来继续分析Okhttp如何进行网络请求,并获取响应对象.

```java
Response result = getResponseWithInterceptorChain();
```

``getResponseWithInterceptorChain`` 才是真正执行网络请求并获得Response对象,看一下具体实现:

```java
#RealCall.java

Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

它首先创建了一个空的List,并把我们用户自定义的拦截器添加进去,然后添加``Okhttp`` 预先定义的拦截器,主要有以下几个:

- ``RetryAndFollowUpInterceptor`` : 负责失败重试和重定向
- `` BridgeInterceptor`` : 负责处理``http`` 通用请求头和返回头,Cookie
- ``CacheInterceptor`` : 负责处理缓存
- ``ConnectInterceptor`` : 开启和服务器的连接
- ``networkInterceptors`` : 用户定义的网络拦截器,权限更高,更接近底层,一般不需要关注
- ``CallServerInterceptor`` : 实现网络请求

这里只是大概介绍每个拦截器的职责,不作详细解析,有兴趣可以自己查看相关源代码.

Okhttp实际上在这里把这些拦截器进行了排序,任何一个顺序改变,都可能导致无法正常工作.

接下来看一下okhttp是如何将这些拦截器串联在一起工作的:

在处理完所有拦截器之后,它创建了一个``RealInterceptorChain`` 对象,并把拦截器列表作为参数传了进去,然后调用了``proceed`` 方法.

```java
#RealInterceptorChain.java

  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

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

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

这里又调用了重载的proceed方法,略过一些参数校验代码,可以看到又创建了一个RealInterceptorChain对象,把之前传过来的参数原封不动的传进去,注意index变了,第一次创建的时候穿的是0,现在传的的index+1,也就是1,然后再获取拦截器列表的第一个拦截器,并调用intercept方法,到此为止责任链设计完成.

其实就是利用的递归的思想,在每个拦截器的内部调用chain.proceed方法,传递给下一个Interceptor,只要最后有一个拦截器没有调用chain.proceed方法,并返回Response对象,则结束递归,依次返回Response到Interceptor,用以对返回结果处理.



每个拦截器都有自己的职责,可以处理从上游传过来的请求对象,也可以处理从下游返回的请求对象,设计相当优雅.

  

  

  

  

  

  

  

  



![](https://upload-images.jianshu.io/upload_images/5713484-fd5ed2418c5c03ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
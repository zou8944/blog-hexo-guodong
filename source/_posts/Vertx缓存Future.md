---
title: Vertx缓存Future
date: 2020-02-27 21:58:08
tags:
 - Vertx
categories:
 - Vertx
---

# 引子

这是一个晴朗的午后，我沐浴着窗口洒落的阳光，懒洋洋地敲着代码，喝着并不存在的咖啡，听着窗外并不存在的熙熙攘攘。这是一个疫情中的午后，深圳二月份的天气算是比较厚道，一件薄外套已经让我微微出汗。我，又遇到bug了，调了一上午的bug，自己写的bug，查了半天的bug，甚至让我分不清此刻的汗水是气温还是bug导致的。

随着时间的流逝，bug终究会解决，我们要做的，就是静静地等着。不知不觉已经到了晚上，果然，bug解决了。往往一个bug的持续时间决定了它是否值得被记录。解决完这个bug时，我惊喜地意识到又可以水一篇博文了。呵呵。

<!-- more -->

在Vertx中，Future是遵循Promise/Future原则的接口，是一个占位符。按官方说的，它代表了一个可能已经发生、或可能还没发生的动作的结果，即一个异步结果。读取其中的结果，通常是设置一个回调方法，但是注意，**一个future只能设置一个回调方法，即一个Handler，或者更具体地说，如果设置多个Handler，则只有最后一个Handler有效。**

```kotlin
val future = Promise.promise<String>().future();
future.setHandler { ar -> 
    if(ar.failed()){
        // 处理失败的情况
    } else {
        // 处理成功的情况
    }
}
```

# 现场回放

## 使用场景

有一缓存需求：将一段读取数据库的代码的结果缓存起来，缓存有效期十分钟，过期后自动刷新，要求整个过程全异步。

于是如下缓存接口

```kotlin
interface Cache<K, V> {

    // get方法，第一个参数为key，第二个参数为缓存过期时获取新的缓存的方法
  fun get(key: K, mappingFunction: () -> Future<V>): Future<V>

  // 删除缓存值
  fun invalidate(key: K)
}
```

使用Caffeine实现上述接口

```kotlin
class CaffeineProxy<K, V>  : Cache<K, V> {

  private val cache: Cache<K, Single<V>> = Caffeine.newBuilder().build()

  override fun get(key: K, mappingFunction: () -> Future<V>): Future<V> = 
    cache.get(key) { mappingFunction.invoke() }

  override fun invalidate(key: K) = cache.invalidate(key!!)

}
```

在协程上下文中使用，如下

```kotlin
class ServiceImpl {
    
    private val locationCache = LocationCache()
    
    // 由于只缓存一段代码的执行结果，因此只有一个key，用一个内部类将缓存包裹起来
    inner class LocationCache {
    // 创建缓存实例
    private val innerCache = CaffeineProxy<String, List<Location>>()
    // 取值方法，取的结果是Future实例
    override suspend fun getCache(): Future<List<JsonObject>> = innerCache.get("UniqueCache") {
        val promise = Promise.promise<List<JsonObject>>()
		adminDao.getAvailableLocations(promise)
        promise.future()
    }
  }
    
  // 在方法1中使用该缓存
  suspend fun fun1() {
      val result = locationCache.getCache().await()
      . . .  后续操作 . . .
  }
    
  // 在方法2中使用该缓存
  suspend fun fun2() {
      val result = locationCache.getCache().await()
      . . .  后续操作 . . .
  }
}
```

## 问题复现

并发较高的场景下，会出现部分方法调用无响应的情况。上述缓存方法放在Web代码中，对应的就是多个会用到缓存的请求同时发起时，部分请求会永远无响应，或者触发系统的超时机制。

## 原因分析

上述缓存有一个大前提，即将Future缓存起来，并在之后的流通中反复使用同一个被缓存的Future。前文中，我们在协程上下文中调用了await()方法，该方法定义如下。

```kotlin
/**
 * Awaits the completion of a future without blocking the event loop.
 */
suspend fun <T> Future<T>.await(): T = when {
  succeeded() -> result()
  failed() -> throw cause()
  else -> suspendCancellableCoroutine { cont: CancellableContinuation<T> ->
    setHandler { asyncResult ->
      if (asyncResult.succeeded()) cont.resume(asyncResult.result() as T)
      else cont.resumeWithException(asyncResult.cause())
    }
  }
}
```

可以看到，其逻辑是：如果成功则返回结果；如果失败则抛出异常；否则调用setHandler()设置回调方法。

再来看看Future的实现了FutureImpl的定义

```java
class FutureImpl<T> implements Promise<T>, Future<T> {

  private boolean failed;
  private boolean succeeded;
  private Handler<AsyncResult<T>> handler;
  private T result;
  private Throwable throwable;
    
    . . . . . .
  /**
   * Set a handler for the result. It will get called when it's complete
   */
  public Future<T> setHandler(Handler<AsyncResult<T>> handler) {
    boolean callHandler;
    synchronized (this) {
      callHandler = isComplete();
      if (!callHandler) {
        this.handler = handler;
      }
    }
    if (callHandler) {
      handler.handle(this);
    }
    return this;
  }
    . . . . . .
}
```

可以看到，setHandler()会将传入的handler直接覆盖掉现有的handler属性。

于是可以分析出正常情景和异常情景如下

- 异常情景

  缓存已过期，此时方法1调用获取缓存方法，拿到Future，此时该Future未完成，于是通过await()方法调用setHandler()设置了一个回调方法；在Future尚未完成前，方法2也调用获取缓存方法，得到同一个Future实例，同样，由于它未完成，于是通过await()方法再次调用setHandler()设置了新的回调方法。

  这样，方法2设置的回调方法覆盖了方法1设置的回调，当Future完成时，方法2的回调方法将得到通知，使得方法2能够正常继续执行；方法1的回调则会永远等待被回调，直到超时。

- 正常情景

  缓存有效，且Future已完成，根据await()方法的定义：先同步地读取Future的结果，在本场景中，一直能够读取到Future结果，而不会进入到setHandler()，这样无论并发多高，都能够正确返回结果。

如果缓存时间设置很长，Future从创建到完成的时间很短，在单元测试阶段甚至SIT都很难发现。很容易可能造成线上偶现的bug，并且相当地隐晦。

## 就这么完了？

到这里，原因找到了，问题修复了，似乎就OK了。但是仔细想想，从语义上，Future代表一个异步执行的结果，常规的使用方法是setHandler()设置回调方法，那一个结果被多处使用似乎是很自然的需求，Vertx设置这样一个限制，是不是有些反直觉，或者反人类呢？

或许我们可以在这个[issue](https://github.com/eclipse-vertx/vert.x/issues/1920)找到些许解释。简而言之，官方回复Future就这样了，如果需要一次生成多次使用，请考虑使用其它库来实现这样的效果，比如RxJava。

我想吐槽的点在于，目前Vertx的Future实现不修改没有问题，但做一些针对上述问题的防护措施也是可以的，只可惜并没有。

# 可被多次消费的占位符



# 引申


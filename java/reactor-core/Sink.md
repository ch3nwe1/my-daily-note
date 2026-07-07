
## MonoSink

```java
void success(T value)
```
> 向管道正向发射一个非空的具体数据。
> 它会瞬间触发下游订阅者的 `onNext(value)`，紧接着触发 `onComplete()` 结束流。
> **注意点**：如果传入 `null`，它的行为等同于调用不带参数的 `success()`（即发射一个空流）。

```java
void success()
```
> 发射一个**空流终结信号**（相当于触发下游的 `onComplete()`），不带任何数据。

```java
void error(Throwable e)
```
> 向下一路引爆，发射一个**错误终结信号**（触发下游的 `onError()`）

```java
ContextView contextView()
```
> 获取下游一路逆流传上来的 **响应式上下文（Context）**。

```java
MonoSink<T> onCancel(Disposable d)
```
> 注册一个回调函数。**只有当下游主动发起 `Subscription.cancel()时**，这个回调才会被触发。
```java
MonoSink<T> onDispose(Disposable d)
```
> **终极全能清理王。** 不管流是因为什么原因结束的——无论是正常放完（`success`）、中途报废（`error`）、还是下游退订（`cancel`），**只要流一死，这个回调必触发！**
```java
MonoSink<T> onRequest(LongConsumer consumer)
```
> 监听下游的**索要信号**。当最终的消费者调用 `request(n)` 的那一瞬间，这个 Lambda 会被立刻触发。



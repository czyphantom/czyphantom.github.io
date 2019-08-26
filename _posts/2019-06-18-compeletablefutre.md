---
layout:     post
title:      CompletableFutre类
subtitle:   
date:       2019-06-18
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

在Java8之前的版本Future接口的实现类只有FutureTask类，但是只提供轮询或者阻塞获取结果的方式，不能及时的获取结果。在Java8中引入了CompletaleFuture类，供了非常强大的Future的扩展功能，可以帮助我们简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合CompletableFuture的方法。

可以通过CompletableFuture的supplyAsync方法创建一个异步计算，当然也可以通过传入一个Executor来执行异步程序，如果不传入默认使用ForkJoinPool.commonPool()。异步计算结束之后还可以指定回调函数，使用supplyAsync().thenAccept()来添加回调。

CompletableFuture另外实现了CompletionStage接口，这个接口代表任务完成的阶段，可以通过实现多个CompletionStage命令，并且将这些命令串联在一起的方式实现多个命令之间的触发。因为这个特性，CompletableFuture具有非常丰富的用法。

+ 操作上一步的结果

```java
//处理完上一步的结果且有返回值
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn,Executor executor);

//处理完上一步的结果没有返回值
public CompletionStage<Void> thenAccept(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```

+ 不关心上一步的结果，直接进行下一步操作

```java
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```

+ 组合两个CompletionStage的结果

```java
//下面三种有返回值
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);

//这下面三种没有返回值，只消耗上一步的结果
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action,     Executor executor);
```

+ 等两个CompletionStage都执行完再执行

```java
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

+ 两个CompletionStage，谁先执行完，就用谁的结果进行下一步操作

```java
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);

public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

+ 两个CompletionStage，任何一个完成了都会执行下一步的操作

```java
public CompletionStage<Void> runAfterEither(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

+ 运行时出现了异常的补偿

```java
public CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);
```

+ 运行完成时，对结果的记录和处理

```java
//运行完成只处理不负责重新返回新的结果
public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action,Executor executor);

//运行完成负责处理，例如抛出异常返回某个值
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```
+ 任意一个返回或者所有都返回

```java
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> allOf(CompletableFuture<?>... cfs)
```

# 参考文章
[CompletableFuture 详解](https://www.jianshu.com/p/6f3ee90ab7d3)
[通过实例理解 JDK8 的 CompletableFuture](https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html)
![image-20221224202032930](E:\img\image-20221224202032930.png)



![image-20221224202454220](E:\img\image-20221224202454220.png)

> FutureTask继承了Runable,同时构造注入了Callable，因此满足其多线程、有返回值、异步三个特点。

代码架构：

![image-20221224204028172](E:\img\image-20221224204028172.png)



- 1.FutrueTask中的get方法容易造成线程阻塞

![image-20221224205728735](E:\img\image-20221224205728735.png)

```java
public class FutrueApiDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<String>futureTask = new FutureTask<String>(()->{
            System.out.println("--come in");
            Thread.sleep(2000);
            return "task over";
        });
        Thread t1 = new Thread(futureTask,"t1");
        t1.start();
        System.out.println(Thread.currentThread().getName()+"线程忙其他事情去了");
        System.out.println(futureTask.get());
    }
}
```



- .提供了get(long millions,TimeUnit unit)的方法可以满足过时不候

```java
public class FutrueApiDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        FutureTask<String>futureTask = new FutureTask<String>(()->{
            System.out.println("--come in");
            Thread.sleep(2000);
            return "task over";
        });
        Thread t1 = new Thread(futureTask,"t1");
        t1.start();
        System.out.println(Thread.currentThread().getName()+"线程忙其他事情去了");
        System.out.println(futureTask.get(1, TimeUnit.SECONDS));
    }
}
```

![image-20221224210653832](E:\img\image-20221224210653832.png)

- 2.轮询容易导致CPU轮询

![image-20221224212652544](E:\img\image-20221224212652544.png)

> 综上可以看出Futrue对获取结果实则是不够友好的

### CompleteFuture对Future的改进 

![image-20221224214304942](E:\img\image-20221224214304942.png)

## CompleteableFuture创建方法

```java
 public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the given executor with the value obtained
     * by calling the given Supplier.
     *
     * @param supplier a function returning the value to be used
     * to complete the returned CompletableFuture
     * @param executor the executor to use for asynchronous execution
     * @param <U> the function's return type
     * @return the new CompletableFuture
     */
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor) {
        return asyncSupplyStage(screenExecutor(executor), supplier);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the {@link ForkJoinPool#commonPool()} after
     * it runs the given action.
     *
     * @param runnable the action to run before completing the
     * returned CompletableFuture
     * @return the new CompletableFuture
     */
    public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the given executor after it runs the given
     * action.
     *
     * @param runnable the action to run before completing the
     * returned CompletableFuture
     * @param executor the executor to use for asynchronous execution
     * @return the new CompletableFuture
     */
    public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor) {
        return asyncRunStage(screenExecutor(executor), runnable);
    }
```

- runAsync方法不支持返回值。其中Executor指的是可以传入我们的线程池对象
- supplyAsync可以支持返回值。其中Executor指的是可以传入我们的线程池对象

### 四大静态方法

- 无返回值

```java
public class CompleteFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(3);
        CompletableFuture<Void>completableFuture = CompletableFuture.runAsync(()->{
            System.out.println(Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },service);
        //获取结果
        System.out.println(completableFuture.get());
    }
}
```



----

![image-20221224220312073](E:\img\image-20221224220312073.png)

---

![image-20221224220427946](E:\img\image-20221224220427946.png)

- 有返回值

```java
public class CompleteFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(3);
        CompletableFuture<Void>completableFuture = CompletableFuture.runAsync(()->{
            System.out.println(Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },service);
        //获取结果
        System.out.println(completableFuture.get());
    }
}
```

### CompletableFuture回调方法

#### whenComplete

```java
    public CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(null, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(asyncPool, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action, Executor executor) {
        return uniWhenCompleteStage(screenExecutor(executor), action);
    }
```

whenComplete可以处理正常的计算结果

> whenComplete和whenCompleteAsync的区别：
>
> whenComplete是当前线程执行当前任务，等待任务执行完成后继续执行当前的whenComplete
>
> whenCompleteAsync：是执行把whenCompleteAysnc这个任务交给线程池中的其他线程来进行执行
>
> 方法中不以Aysnc，代表着Action使用相同的线程来进行操作
>
> 方法以Aysnc结尾可能会使用其他线程执行（如果是使用相同的线程池，也可能会被同一个线程选中执行）

```java
/**
 * @Author: sakura
 * @Date: 2022/12/26 9:28
 * @Description: 异步编程的思想
 * @Version 1.0
 */
public class CompleteFutureDemo2 {
    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(3);
        try {
            CompletableFuture<Integer>future = CompletableFuture.supplyAsync(()->{
                System.out.println(Thread.currentThread().getName()+"--come-in");
                int result = ThreadLocalRandom.current().nextInt(10);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("---2秒后计算结果为："+result);
                return result;
            },service).whenComplete((v,e)->{
                if(e==null){
                    System.out.println("计算完成，更新系统updateValue"+v);
                }
            }).exceptionally(e->{
                e.printStackTrace();
                System.out.println("异常情况"+e.getCause()+"\t"+e.getMessage());
                return null;
            });

            System.out.println(Thread.currentThread().getName()+"主线程忙其他任务去了");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            service.shutdown();
        }
    }
}
```

#### handle

hanlde: whenComplete和execptionally结合版。方法执行后的处理，无论成功与失败都可以处理。

```java
 public <U> CompletableFuture<U> handle(
        BiFunction<? super T, Throwable, ? extends U> fn) {
        return uniHandleStage(null, fn);
    }

    public <U> CompletableFuture<U> handleAsync(
        BiFunction<? super T, Throwable, ? extends U> fn) {
        return uniHandleStage(asyncPool, fn);
    }

    public <U> CompletableFuture<U> handleAsync(
        BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) {
        return uniHandleStage(screenExecutor(executor), fn);
    }
```

案例：

```java
CompletableFuture<Integer>completableFuture2 = CompletableFuture.suupplyAysnc(()->{
    System.out.println(Thread.current.getName());
    Sytem.out.println("CompletableFuture...");
    return 10/0;
}.service).handle((t,u)->{
    if(t!=null){
        System.out.println("存在返回结果："+t);
        return 8;
    }
    if(u!=null){
        Sytem.out.println("存在异常常："+u);
        return 9;
    }
    return 5;
});
System.out.println(completableFuture.get())
    
```



### 异步编程



![image-20221226094311662](../img/image-20221226094311662.png)

```java
public class CompleteFutureDemo2 {
    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(3);
        try {
            CompletableFuture<Integer>future = CompletableFuture.supplyAsync(()->{
                System.out.println(Thread.currentThread().getName()+"--come-in");
                int result = ThreadLocalRandom.current().nextInt(10);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("---2秒后计算结果为："+result);
                return result;
            },service).whenComplete((v,e)->{
                if(e==null){
                    System.out.println("计算完成，更新系统updateValue"+v);
                }
            }).exceptionally(e->{
                e.printStackTrace();
                System.out.println("异常情况"+e.getCause()+"\t"+e.getMessage());
                return null;
            });

            System.out.println(Thread.currentThread().getName()+"主线程忙其他任务去了");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            service.shutdown();
        }
    }
}
```

### 函数式编程接口小总结

![image-20221226094749274](../img/image-20221226094749274.png)

### 电商比价案列

> 需求查询一个商品，需要查到每个网站上不同的价格，进行比价，使用函数式编程及stream流
>
> - 1、step by step
> - 2、使用CompleteFuture接口进行我们的异步线程，可以提高性能

```java
public class CompletableFutureNetMallDemo {
    static List<NetMall> list = Arrays.asList(
            new NetMall("jd"),
            new NetMall("pdd"),
            new NetMall("taobao"),
            new NetMall("dangdangwang"),
            new NetMall("tmall"));

    //同步 ,step by step
    /**
     * List<NetMall>  ---->   List<String>
     * @param list
     * @param productName
     * @return
     */
    public static List<String> getPriceByStep(List<NetMall> list,String productName) {

        return list
                .stream()
                .map(netMall -> String.format(productName + " in %s price is %.2f", netMall.getMallName(),
                                netMall.calcPrice(productName)))
                .collect(Collectors.toList());
    }
    //异步 ,多箭齐发
    /**
     * List<NetMall>  ---->List<CompletableFuture<String>> --->   List<String>
     * @param list
     * @param productName
     * @return
     */
    
    public static List<String> getPriceByASync(List<NetMall> list,String productName) {
        return list
                .stream()
                .map(netMall ->
                        CompletableFuture.supplyAsync(() ->
                        String.format(productName + " is %s price is %.2f", netMall.getMallName(), netMall.calcPrice(productName)))
                )
                .collect(Collectors.toList())
                .stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
    }

    public static void main(String[] args) {

        long startTime = System.currentTimeMillis();
        List<String> list1 = getPriceByStep(list, "mysql");
        for (String element : list1) {
            System.out.println(element);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒");

        System.out.println();

        long startTime2 = System.currentTimeMillis();
        List<String> list2 = getPriceByASync(list, "mysql");
        for (String element : list2) {
            System.out.println(element);
        }
        long endTime2 = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime2 - startTime2) +" 毫秒");

    }
}

@Data
@AllArgsConstructor
class NetMall {
    private String mallName;
    public double calcPrice(String productName) {
        //检索需要1秒钟
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
        return ThreadLocalRandom.current().nextDouble() * 2 + productName.charAt(0);
    }
}
/*
	mysql in jd price is 110.59
	mysql in pdd price is 110.23
	mysql in taobao price is 110.04
	mysql in dangdangwang price is 110.08
	mysql in tmall price is 109.91
	----costTime: 5030 毫秒
	mysql is jd price is 109.07
	mysql is pdd price is 109.47
	mysql is taobao price is 109.04
	mysql is dangdangwang price is 110.09
	mysql is tmall price is 110.72
	----costTime: 2026 毫秒
**/
```

- 防止主线程先结束，默认的Compelete的线程池是守护线程，可以采取睡眠一会，或者是创建线程池，调用ThreadPool.shutdown进行关闭

## CompletableFuture异步任务场景

### 线程串行化

```java
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}

public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) {
    return uniAcceptStage(asyncPool, action);
}

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action,
                                               Executor executor) {
    return uniAcceptStage(screenExecutor(executor), action);
}

public CompletableFuture<Void> thenRun(Runnable action) {
    return uniRunStage(null, action);
}

public CompletableFuture<Void> thenRunAsync(Runnable action) {
    return uniRunStage(asyncPool, action);
}

public CompletableFuture<Void> thenRunAsync(Runnable action,
                                            Executor executor) {
    return uniRunStage(screenExecutor(executor), action);
}
```

- thenRun : 不能获取到上一步的结果值，也无返回值
- thenAccept:有结果值，但是无返回值
- thenApplyAysnc:有上一步的结果值，也有返回值



![image-20221226170630901](../img/image-20221226170630901.png)

### 合并结果，双线程

```java
    public <U,V> CompletableFuture<V> thenCombine(
        CompletionStage<? extends U> other,
        BiFunction<? super T,? super U,? extends V> fn) {
        return biApplyStage(null, other, fn);
    }

    public <U,V> CompletableFuture<V> thenCombineAsync(
        CompletionStage<? extends U> other,
        BiFunction<? super T,? super U,? extends V> fn) {
        return biApplyStage(asyncPool, other, fn);
    }

    public <U,V> CompletableFuture<V> thenCombineAsync(
        CompletionStage<? extends U> other,
        BiFunction<? super T,? super U,? extends V> fn, Executor executor) {
        return biApplyStage(screenExecutor(executor), other, fn);
    }
    public <U> CompletableFuture<Void> thenAcceptBoth(
        CompletionStage<? extends U> other,
        BiConsumer<? super T, ? super U> action) {
        return biAcceptStage(null, other, action);
    }

    public <U> CompletableFuture<Void> thenAcceptBothAsync(
        CompletionStage<? extends U> other,
        BiConsumer<? super T, ? super U> action) {
        return biAcceptStage(asyncPool, other, action);
    }

    public <U> CompletableFuture<Void> thenAcceptBothAsync(
        CompletionStage<? extends U> other,
        BiConsumer<? super T, ? super U> action, Executor executor) {
        return biAcceptStage(screenExecutor(executor), other, action);
    }

    public CompletableFuture<Void> runAfterBoth(CompletionStage<?> other,
                                                Runnable action) {
        return biRunStage(null, other, action);
    }

    public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other,
                                                     Runnable action) {
        return biRunStage(asyncPool, other, action);
    }

    public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other,
                                                     Runnable action,
                                                     Executor executor) {
        return biRunStage(screenExecutor(executor), other, action);
    }
```

- runAfterBothAsync两人任务组合，不能得到前任务的结果和无返回值
- thenAccpetBothAysnc两人任务组合，能得到前任务的结果和无返回值
- thenCombineAysnc两人任务组合，能得到前任务的结果和有返回值

` 传入的参数 ` ：CompletableStage是我们的CompleteFuture

两个任务必须完成，触发该Runable参数指定的任务即当前lambada表达式的内容

**thenCombine**:组合两个future,获取两个future的返回结果，并返回当前任务的返回值

thenAcceptBoth：组合两个future，获取两个future的返回结果，并返回当前任务的返回值

runAfterBoth:组合两个future,不需要获取future的结果，只需要两个future处理完任务后处理该任务

![image-20221226172303016](../img/image-20221226172303016.png)

![image-20221226172346860](../img/image-20221226172346860.png)

![image-20221226172426153](../img/image-20221226172426153.png)

### 双线程完成其中一个就可以：

```JAVA
 public <U> CompletableFuture<U> applyToEither(
        CompletionStage<? extends T> other, Function<? super T, U> fn) {
        return orApplyStage(null, other, fn);
    }

    public <U> CompletableFuture<U> applyToEitherAsync(
        CompletionStage<? extends T> other, Function<? super T, U> fn) {
        return orApplyStage(asyncPool, other, fn);
    }

    public <U> CompletableFuture<U> applyToEitherAsync(
        CompletionStage<? extends T> other, Function<? super T, U> fn,
        Executor executor) {
        return orApplyStage(screenExecutor(executor), other, fn);
    }

    public CompletableFuture<Void> acceptEither(
        CompletionStage<? extends T> other, Consumer<? super T> action) {
        return orAcceptStage(null, other, action);
    }

    public CompletableFuture<Void> acceptEitherAsync(
        CompletionStage<? extends T> other, Consumer<? super T> action) {
        return orAcceptStage(asyncPool, other, action);
    }

    public CompletableFuture<Void> acceptEitherAsync(
        CompletionStage<? extends T> other, Consumer<? super T> action,
        Executor executor) {
        return orAcceptStage(screenExecutor(executor), other, action);
    }

    public CompletableFuture<Void> runAfterEither(CompletionStage<?> other,
                                                  Runnable action) {
        return orRunStage(null, other, action);
    }

    public CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other,
                                                       Runnable action) {
        return orRunStage(asyncPool, other, action);
    }

    public CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other,
                                                       Runnable action,
                                                       Executor executor) {
        return orRunStage(screenExecutor(executor), other, action);
    }
```

- RunAfterEither:两个任务有一个执行完成，不需要获取future的结果，处理任务，也没有返回值。
- acceptEither:两个任务有一个执行完成，获取它的返回值，处理任务，没有新的返回值，只能返回调用acceptEitherAsync函数
- applyToEither:两个任务有一个执行完成，获取它的返回值，处理任务并有新的返回值。

![image-20221226173105059](../img/image-20221226173105059.png)

![image-20221226173205438](../img/image-20221226173205438.png)

![image-20221226173216052](../img/image-20221226173216052.png)

### 多任务组合

```java
    public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
        return andTree(cfs, 0, cfs.length - 1);
    }

    /**
     * Returns a new CompletableFuture that is completed when any of
     * the given CompletableFutures complete, with the same result.
     * Otherwise, if it completed exceptionally, the returned
     * CompletableFuture also does so, with a CompletionException
     * holding this exception as its cause.  If no CompletableFutures
     * are provided, returns an incomplete CompletableFuture.
     *
     * @param cfs the CompletableFutures
     * @return a new CompletableFuture that is completed with the
     * result or exception of any of the given CompletableFutures when
     * one completes
     * @throws NullPointerException if the array or any of its elements are
     * {@code null}
     */
    public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {
        return orTree(cfs, 0, cfs.length - 1);
    }
```

- `allOf`:等待所有任务完成
- `anyOf`:只要有一个任务完成

![image-20221226173249361](../img/image-20221226173249361.png)

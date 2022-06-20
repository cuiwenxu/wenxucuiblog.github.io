- 第一阶段：无返回值（继承thread或实现runnable）
优点：代码简单
缺点：
Runnable接口有个问题，它的方法没有返回值。如果任务需要一个返回结果，那么只能保存到变量，还要提供额外的方法读取，非常不便

- 第二阶段：有返回值，阻塞主线程（实现Callable）
优点：有返回值，返回结果为Future
缺点：调用future.get()时会阻塞主线程。

当我们提交一个Callable任务后，我们会同时获得一个Future对象，然后，我们在主线程某个时刻调用Future对象的get()方法，就可以获得异步执行的结果。在调用get()时，如果异步任务已经完成，我们就直接获得结果。如果异步任务还没有完成，那么get()会阻塞，直到任务完成后才返回结果。
```java
ExecutorService executor = Executors.newFixedThreadPool(4); 
// 定义任务:
Callable<String> task = new Task();
// 提交任务并获得Future:
Future<String> future = executor.submit(task);
// 从Future获取异步执行返回的结果:
String result = future.get(); // 可能阻塞
```
- 第三阶段：有返回值且不阻塞主线程（CompletableFuture）
优点：不阻塞主线程
```java
CompletableFuture<Double> cf = CompletableFuture.supplyAsync(Main::fetchPrice);
// 如果执行成功:
cf.thenAccept((result) -> {
            System.out.println("price: " + result);
});
// 如果执行异常:
cf.exceptionally((e) -> {
    e.printStackTrace();
    return null;
});
// 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
Thread.sleep(200);
```

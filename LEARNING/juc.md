## Future

## CompletableFuture

* runAsync无返回值 (public static CompletableFuture `<Void> runAsync(Runnable r, Executor e)`)
* supplyAsync有返回值(public statuc `<U> CompletableFuture<U> supplyAsync(Supplier<U> supplier， Executor e)`)
* 未提供线程池时，使用ForkJoinPool.commonPool作为默认线程池（如果核心线程数小于1，则无法使用ForkJoinPool，此时将使用ThreadPerTaskExecutor保证用户的任务一定执行）
* 常用回调
  * thenApply(s -> s)：接受前序结果，执行Function
  * thenRun(() -> return;)：任务完成后执行Runnable
  * thenAccept(s -> return;)：接收前序结果，执行Consumer
* 组合多个异步任务
  * thenCompose：串行依赖
    * 例子：CompletableFuture.supplyAsync(() -> 10).thenCompose(userId -> CompletableFuture.supplyAsync(() -> "userId:" + userId));
  * thenCombine：并行执行，合并结果
    * 例子：future1.thenCombine(future2,  Integer::sum);
  * allOf/anyOf：多任务聚合
    * allOf：等待所有任务完成（无返回值，许手动获取每个结果）
      * 例子：CompletableFuture `<Void> allDone = `CompleteableFuture.allOf(future1, future2, future3); allDone.thenRun(() -> {});
    * anyOf：任意一个任务完成就返回（返回第一个完成的结果）
* 异常处理
  * exceptionally：捕获前序任务的异常，返回默认值
  * handle：无论成功/失败都执行，同时处理结果和异常

## ForkJoinPool

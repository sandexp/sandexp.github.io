#### 提交任务

1. 提交无返回值任务

   ```java
   runAsync(Runnable);
   ```

2. 提交有返回值任务

   ```java
   supplyAsync(Supplier);
   ```

#### 回调函数

`CompletableFuture` 可以在结果上面再加一个callback，当得到结果之后，再接着执行callback。

常用的回调函数有如下三类:

1. `thenRun`后面跟的是一个无参数、无返回值的方法
2. `thenAccept` 后面跟的是一个有参数、无返回值的方法
3.  `thenApply` 后面跟的是一个有参数、有返回值的方法

#### 组合逻辑

1. then compose

   用于`CompletableFuture`的嵌套

2. then combine

   进行多个Future的复合运算


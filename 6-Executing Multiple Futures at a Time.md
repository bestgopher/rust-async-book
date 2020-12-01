到现在为止，我们主要通过`.await`执行`Futures`，它阻塞当前的任务知道一个特定的`Future`完成。然而，真正给你的异步程序通常需要并行执行多个不同的任务。

在这章，我们将会介绍一些同时执行多个异步操作的方法：

- `join!`：等待`futures`所有的都完成
- `select!`：等待多个`futures`中其中一个完成
- `spawning`：创建一个高级别的任务，运行future直到完成
- `FuturesUnordered`：产生每个子`future`的一组`futures`。


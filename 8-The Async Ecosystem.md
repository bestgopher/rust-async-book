Rust目前仅提供编写异步代码的基本知识。`executors`、`tasks`、`reactors`、`combinators`和一些底层的IO futures和traits在标准库中并没有提供。但是同时，社区提供的异步生态系统填补了这些空白。



### Async Runtimes

异步运行时是用来执行异步程序的库。运行时通常捆绑一个`reactor`到一个或者多个`exectors`上。`reactor`为异步IO、进程间通信和定时器等外部事件提供了订阅机制。在一个异步运行时中，订阅者通常代表的是底层的IO操作。`executor`负责调度和执行`task`。它们跟踪运行或暂停的`task`，轮询`future`来完成它们，并在任务取得进展时唤醒任务。`executor`一词经常与`runtime`互换使用。在这里，我们使用`ecosystem`一词来描述捆绑了兼容`trait`和功能的`runtime`。



### Community-Provided Async Crates

#### The Futures Crate

[future crate](https://docs.rs/futures/) 包含用于编写异步代码的traits和函数。包含了`Stream`、`Sink`、`AsyncRead`和`AsyncWrite` traits，和`combinators`这样的公共组件。这些公共组件和traits可能最终会成为标准库的一部分。

`futures`有它自己的`executor`，但是没有`reactor`，因此它不能支持异步IO或者定时的特性。因为这个，它不能视为一个完整的运行时。一个通常的选择是`futures`的公共组件和其它crate的`executor`配合使用。



#### Popular Async Runtimes

标准库没有异步的运行时，也没有官方推荐的。下面的crates提供了一些广受欢迎的运行时。

- [Tokio](https://docs.rs/tokio/)：一个受欢迎的异步社区，包括HTTP，gRPC和tracing框架。
- [async-std](https://docs.rs/async-std/)：一个提供了标准库组件副本的crate。
- [smol](https://docs.rs/smol/)：一个小巧、简化的异步运行时。提供`Async` trait，可以用来包装像`UnixStream`或者`TcpListener`这样的结构体。
- [fuchsia-async](https://fuchsia.googlesource.com/fuchsia/+/master/src/lib/fuchsia-async/)：在Fuchsia os中使用的executor。



### Determining Ecosystem Compatibility

不是所有的异步程序、框架和库是与其它库或者操作系统或者平台兼容的。大多数异步代码可以在任何生态系统(Ecosystem)中使用，但是一些框架和库需要特定的生态系统。生态系统的限制并不总是记录在案的，但是有一些经验法则来断定这个库、trait或者函数是否依赖一个特定的生态系统。

任何与异步IO、计时器、进程间通信，或者任务的异步代码通常依赖特定的异步`executor`或者`reactor`。而其它类型的代码，例如异步表达式、`combinators`、同步类型、流，通常与生态系统无关的，这些代码提供的任何嵌套的`future`也与生态系统无关。在开始一个项目前，推荐去搜寻相关的异步框架和库，并确保与你选择的运行时兼容。

值得注意的是，`Tokio`使用`mio`的`reactor`，定义它自己版本的异步IO traits，包括`AsyncRead`和`AsyncWrite`。

`Tokio`不与`async-std`和`smol`兼容，`async-std`和`smol`依靠[`async-executor` crate](https://docs.rs/async-executor)和定义在`futures`中的`AsyncRead`和`AsyncWrite` traits。

冲突的运行时需求有时可以通过兼容层解决，这些兼容层运行你在一个运行时中调用为另一个运行时写的代码。例如，[`async_compat` crate](https://docs.rs/async_compat) 提供了一个`Tokio`和其它运行时的兼容层。

库暴露的异步API不应该依赖一个特定的`executor`或者`reactor`，除非它们需要生产tasks或者定义它们自己的异步IO或计时器 futures。理想情况下，只有二进制程序负责调度和运行tasks。



### Single Threaded vs Multi-Threaded Executors

异步`executors`可以是单线程或者多线程的。例如，`async-executor` crate 有一个单线程的`LocalExecutor`和一个多线程的`Executor`。

一个多线程的`executor`可以同时进行多个任务。对于具有许多任务的工作来说，它可以大大加快执行速度，但是在任务之间同步数据通常会更昂贵。建议度量你程序的性能来决定选择单线程还是多线程运行时。

任务可以在创建它们的线程上运行，也可以在另外的线程上运行。异步运行时通常提供将任务生成到单独线程上的功能。即使任务在另外的线程上执行，它们仍然是非阻塞的。为了在多线程执行器上调度任务，它们必须是`Send`的。有些运行时提供函数生产非`Send`的任务，这些能确保任务在生产它们的线程上执行。它们也提供了在专门的线程上生成阻塞任务，这些对于运行其它库的阻塞同步代码很有用的。




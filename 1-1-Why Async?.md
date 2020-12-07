我们都喜欢rust允许我们编写快速，安全的软件。但是为什么要编写异步代码呢？

异步代码允许我们并发地在相同系统线程上运行多个任务。在典型的线程应用程序中，如果你想要同时下载两个不同网页，你需要把工作分散到两个不同的线程中，就像这样：

```rust
fn get_two_sites() {
    // Spawn two threads to do work.
    let thread_one = thread::spawn(|| download("https://www.foo.com"));
    let thread_two = thread::spawn(|| download("https://www.bar.com"));

    // Wait for both threads to complete.
    thread_one.join().expect("thread one panicked");
    thread_two.join().expect("thread two panicked");
}
```

这对许多程序都适用 - 毕竟线程就是被设计用来做这个的：同时运行多个不同的任务。然而，它们也带来了一些局限性。在不同线程之间的切换和在线程间数据共享的过程设计很多开销。即使一个坐着不执行任何事情的线程也会占用宝贵的系统资源。这些是异步代码旨在消除的成本。我们可以使用Rust的`async/.await`符号重写上面的函数，它们允许我们允许我们同时运行多个任务而不创建多个线程。

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // Run both futures to completion at the same time.
    join!(future_one, future_two);
}
```

总的来说，异步程序相较于相应的多线程实现，有潜力更快，使用更少的资源。然而，这是有成本的。线程是操作系统原生支持的，使用它们不需要任何特定的程序模型 - 任何函数都可以创建一个线程，使用线程调用函数就像调用任何普通函数一样简单。然而，异步函数需要语言或者库的特殊支持，`async fn`创建一个异步函数，它返回一个`Future`。为了执行函数体，返回的`Future`必须运行完毕。

特别需要铭记的是传统的线程程序也是很便利，并且Rust的内存占用量少和可预测性意味着你无需使用`async`就能解决问题。异步编程模型增加的复杂性并不总是值得的，考虑你的程序是否通过使用一个更简单的多线程模型就能更好的运行是非常重要的。
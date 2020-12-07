`async/.await`是rust的内建工具，用来编写看起来像同步代码的异步代码。`async`转换一个代码块到一个实现了`Future` trait的状态机中。鉴于在同步代码中调用一个阻塞函数将会阻塞整个线程，阻塞的`Future`将会让出线程的控制权，允许其他的`Future`运行。

让我们添加一些依赖到`Cargo.toml`文件中：

```rust
[dependencies]
futures = "0.3"
```

创建一个异步函数，你可以使用`async fn`语法：

```rust
async fn do_something() { /* ... */ }
```

`async fn`返回的值是一个`Future`。`Future`需要在一个`executor`上运行。

```rust
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // Nothing is printed
    block_on(future); // `future` is run and "hello, world!" is printed
}
```

在`async fn`内部，你可以使用`.await`等待其他实现了`Future` trait的类型，例如其他`async fn`的输出。与`block_on` 不同，`.await`不会阻塞当前线程，而是异步等待这个future完成，如果future当前没有进展时允许其他任务运行。

例如，假设我们有三个`async fn`：`learn_song`、`sing_song` 和 `dance`：

```rust
async fn learn_song() -> Song { /* ... */ }
async fn sing_song(song:Song) { /* ... */ }
async fn dance() { /* ... */ }
```

一种情况是学习唱歌，唱歌，跳舞分别阻塞地进行：

```rust
fn main() {
    let song = block_on(learn_song());
    block_on(sing_song(song));
    block_on(dance());
}
```

然而，这个方法我们不能尽可能获取最好的性能 - 我们只能同时做一件事！很明显我们必须在唱歌之前学习它，但是能够在学习唱歌和唱歌的时候跳舞。为了达到这个目的，我们可以创建两个分开的`async fn`，这两个可以并发地运行。

```rust
async fn learn_and_sing() {
    // Wait until the song has been learned before singing it.
    // We use `.await` here rather than `block_on` to prevent blocking the
    // thread, which makes it possible to `dance` at the same time.
    let song = learn_song().await;
    sing_song(song).await;
}

async fn async_main() {
    let f1 = learn_and_sing();
    let f2 = dance();

    // `join!` is like `.await` but can wait for multiple futures concurrently.
    // If we're temporarily blocked in the `learn_and_sing` future, the `dance`
    // future will take over the current thread. If `dance` becomes blocked,
    // `learn_and_sing` can take back over. If both futures are blocked, then
    // `async_main` is blocked and will yield to the executor.
    futures::join!(f1, f2);
}

fn main() {
    block_on(async_main());
}
```

在这个例子中，学习唱歌必须在唱歌之前，但是在学习唱歌和唱歌的同时可以跳舞。如果我们在`learn_and_sing`函数中使用的是`block_on(learn_song())`而不是`learn_song().await`，当`learn_song()`运行的时候这个线程将不会做其他任何事。这时将不能同时跳舞。通过`.await` `learn_song` future，当`learn_song`阻塞的时候，我们允许其他任务接管当前线程。这就能使得多个futures在相同线程中并发地完成。


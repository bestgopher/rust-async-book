`Future` trait是rust异步编程的核心。`Future`是一个可以产生值的异步计算(即使这个值可能为空，例如`()`)。一个*简单*版本的`Future` trait可能如下所示：

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

通过调用`poll`函数，`Future`能向前运行，它能驱动`futures`尽可能完成。如果`futures`完成了，它返回`Poll::Ready(result)`。如果这个`future`不能完成，它返回`Poll::Pending`，且当这个`Future`可以取得进展时调用`wake()`函数。当`wake()`被调用时，`executor`驱动这个`Future`再次调用`poll`，以便这个`Future`能够再次前进。

没有`wake()`，当一个特定的`future`取得进展时，`executor`没办法知道，只能不断地轮训每个`future`。通过`wake()`，`executor`知道提取已经准备完毕，可以被`poll`的`future`。

例如，考虑这样的场景：我们想要从一个套接字中读取数据，套接字中可能有，也可能没有可用的数据。如果有数据，我们可以读取它，然后返回`Poll::Ready(data)`，但是如果没有数据可读，我们的`future`阻塞，不能向前进展。当没有数据可用时，我们必须注册`wake`函数，当套接字里的数据准备完毕后，调用这个函数，它将会告诉`executor`我们的`future`准备好再次进展了。一个简单的`SocketRead` 可能像这样：

```rust
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // The socket has data-- read it into a buffer and return it.
            Poll::Ready(self.socket.read_buf())
        } else {
            // The socket does not yet have data.
            // 安排
            // Arrange for `wake` to be called once data is available.
            // When data becomes available, `wake` will be called, and the
            // user of this `Future` will know to call `poll` again and
            // receive data.
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```

`Future`的模型允许将多个异步操作组合在一起而无需中间分配。同时运行多个`Future`或者串联`Future`在一起可以通过零分配状态机实现，像这样：

```rust
/// A SimpleFuture that runs two other futures to completion concurrently.
///
/// Concurrency is achieved via the fact that calls to `poll` each future
/// may be interleaved, allowing each future to advance itself at its own pace.
pub struct Join<FutureA, FutureB> {
    // Each field may contain a future that should be run to completion.
    // If the future has already completed, the field is set to `None`.
    // This prevents us from polling a future after it has completed, which
    // would violate the contract of the `Future` trait.
    a: Option<FutureA>,
    b: Option<FutureB>,
}

impl<FutureA, FutureB> SimpleFuture for Join<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        // Attempt to complete future `a`.
        if let Some(a) = &mut self.a {
            if let Poll::Ready(()) = a.poll(wake) {
                self.a.take();
            }
        }

        // Attempt to complete future `b`.
        if let Some(b) = &mut self.b {
            if let Poll::Ready(()) = b.poll(wake) {
                self.b.take();
            }
        }

        if self.a.is_none() && self.b.is_none() {
            // Both futures have completed-- we can return successfully
            Poll::Ready(())
        } else {
            // One or both futures returned `Poll::Pending` and still have
            // work to do. They will call `wake()` when progress can be made.
            Poll::Pending
        }
    }
}
```

这展示了多个`futures`怎么同时运行而不需要分别分配，从而实现了更加高效的异步程序。类似地，多个顺序的`futures`可以一个接着一个运行，像这样：

```rust
/// A SimpleFuture that runs two futures to completion, one after another.
//
// Note: for the purposes of this simple example, `AndThenFut` assumes both
// the first and second futures are available at creation-time. The real
// `AndThen` combinator allows creating the second future based on the output
// of the first future, like `get_breakfast.and_then(|food| eat(food))`.
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                // We've completed the first future-- remove it and start on
                // the second!
                Poll::Ready(()) => self.first.take(),
                // We couldn't yet complete the first future.
                Poll::Pending => return Poll::Pending,
            };
        }
        // Now that the first future is done, attempt to complete the second.
        self.second.poll(wake)
    }
}
```

这个例子展示了`Future`怎么被用来表达异步控制流，而不需要多个分配的对象，或者不需要深度嵌套的回调。有了基本的控制流，我们谈论下真正的`Future` trait，看看它有什么不同：

```rust
trait Future {
    type Output;
    fn poll(
        // Note the change from `&mut self` to `Pin<&mut Self>`:
        self: Pin<&mut Self>,
        // and the change from `wake: fn()` to `cx: &mut Context<'_>`:
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

你注意到的第一个变化时`self`类型不再是`&mut Self`，而变成了`Pin<&mut Self>`。我们将会在后面的章节讨论`pinning`，现在你只需知道它能帮助我们创建一个不可动的`future`。不可动的对象可以存储它们字段间的指针，例如`struct MyFut {a: i32, ptr_to_a: *const i32}`。固定对与开启`async/await`是必须的。

第二，`wake: fn()` 变成了 `&mut Context<'_>`。在`SimpleFuture`，我们通过调用一个函数指(`fn()`)来告诉`future executor`这个`future`可以被`poll`了。然而，因为`fn()`仅仅是一个函数指针，它不能存储调用`wake`相关`Future`的任何数据。

在现实世界中，一个复杂应用程序，例如一个web服务可以有成千上万不同的链接，这些被唤醒的链接需要单独的管理。`Context`类型通过提供对`Waker`类型的值的访问来解决这个问题，它可以被用来唤醒特定的任务。
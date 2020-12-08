​	`future`在第一次被`poll`时是没完成的，这种情况很常见。当出现这种情况，`future`需要确认它再次被`poll`时它已经可以再次取得进展了。这通过`Waker`类型完成。

`future`每次被`poll`，它被作为任务的一部分被`poll`。tasks是高级别的`future`，它被`executor`提交。

`Waker`提供了一个`wake`方法，这个方法可以用来告诉`executor`关联的任务应该被唤醒。当`wake()`被调用，`executor`知道与`Waker`关联的任务已经准备取得进展，它的`future`可以被再次`poll`。

`Waker`也实现了`clone()`，因此它可以被复制和存储。

让我们用`Waker`来实现一个简单的计时器`future`。



### Applied: Build a Timer

在这个例子中，我们在创建一个计时器时生成一个新的线程，睡眠相应的时间，睡眠时间达到后向计时器`future`发出信号。

这是我们开始需要导入的依赖：

```rust
use std::{
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration,
};
```

开始定义`future`类型。我们的`future`需要有一种方式在线程中传达计时器已经完毕且`future`完成了。我们使用`Arc<Mutex<..>>`的值来达到线程与`future`之间的通信。

```rust
pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

/// Shared state between the future and the waiting thread
struct SharedState {
    /// Whether or not the sleep time has elapsed
    completed: bool,

    /// The waker for the task that `TimerFuture` is running on.
    /// The thread can use this after setting `completed = true` to tell
    /// `TimerFuture`'s task to wake up, see that `completed = true`, and
    /// move forward.
    waker: Option<Waker>,
}
```

现在，我们来编写`Future`的实现：

```rust
impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Look at the shared state to see if the timer has already completed.
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            // Set waker so that the thread can wake up the current task
            // when the timer has completed, ensuring that the future is polled
            // again and sees that `completed = true`.
            //
            // It's tempting to do this once rather than repeatedly cloning
            // the waker each time. However, the `TimerFuture` can move between
            // tasks on the executor, which could cause a stale waker pointing
            // to the wrong task, preventing `TimerFuture` from waking up
            // correctly.
            //
            // N.B. it's possible to check for this using the `Waker::will_wake`
            // function, but we omit that here to keep things simple.
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

相当简单，对吧？如果线程被设置为`shared_state.completed = true`，我们就大功告成了。除此之外，我们为当前任务克隆了`Waker`，然后传递到`shared_state.waker`，以便线程可以唤醒任务。

记住，当`Future`每次被`poll`时，我们都必须更新`Waker`，因为`future`可能被move到不同的任务以及`Waker`。当`future`被`poll`后在不同任务间传递时，这个情况会发生。

最后，我们需要实际构造这个计时器和开始线程的API：

```rust
impl TimerFuture {
    /// Create a new `TimerFuture` which will complete after the provided
    /// timeout.
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // Spawn the new thread
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            // Signal that the timer has completed and wake up the last
            // task on which the future was polled, if one exists.
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```

这就是我们需要构建的简单计时器`future`。现在，如果我们有`executor`来运行这个`future`的话。。。


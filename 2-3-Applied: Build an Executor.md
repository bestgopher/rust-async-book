rust的`Future`是惰性的：它们不会做任何事，除非积极地驱动它去完成。驱动一个`future`完成的一种方式是在`async`函数中`.await`，但这只是把问题向上推而已：谁运行最上层的`async`函数？答案是我们需要一个`Future`执行器。

`Future`执行器获取一系列顶层的`Future`，然后在任何`Future`可以取得进展的时候调用`poll`方法使其运行直到完成。通常，一个执行器将会`poll`一个`future`来开始。当`Future`通过调用`wake()`表面自己准备取得进展的时候，它们被方法一个队列中，然后再次`poll`，重复这个操作直到`Future`运行完毕。

在本节，我们将会编写一个自己的简单执行器，它能并发地允许大量顶级的`future`。

在本例中，我们依赖`futures` crate的`ArcWake` trait，它提供了一个简单方法来构造`Waker`。

```toml
[package]
name = "xyz"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2018"

[dependencies]
futures = "0.3"
```

接下来，我们需要在`src/main.rs`中导入以下依赖：

```rust
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake},
    },
    std::{
        future::Future,
        sync::mpsc::{sync_channel, Receiver, SyncSender},
        sync::{Arc, Mutex},
        task::{Context, Poll},
        time::Duration,
    },
    // The timer we wrote in the previous section:
    timer_future::TimerFuture,
};
```

我们的执行器通过一个通道发送任务去执行。这个执行器将会从通道中拉取事件，然后执行它。当一个任务准备进一步运行时(被唤醒)，它可以调度它自己再次返回通道，然后被`poll`。

在这个设计中，执行器仅仅需要任务通道的接收端。用户获取任务通道的发送端，以便它们能生成新的`future`。任务本身就是`future`，它们能调度它们本身，因此我们存储它们作为一个`future`，并与发送者绑定，这样任务就能重新入队。

```rust
/// Task executor that receives tasks off of a channel and runs them.
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

/// `Spawner` spawns new futures onto the task channel.
#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

/// A future that can reschedule itself to be polled by an `Executor`.
struct Task {
    /// In-progress future that should be pushed to completion.
    ///
    /// The `Mutex` is not necessary for correctness, since we only have
    /// one thread executing tasks at once. However, Rust isn't smart
    /// enough to know that `future` is only mutated from one thread,
    /// so we need to use the `Mutex` to prove thread-safety. A production
    /// executor would not need this, and could use `UnsafeCell` instead.
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    /// Handle to place the task itself back onto the task queue.
    task_sender: SyncSender<Arc<Task>>,
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    // Maximum number of tasks to allow queueing in the channel at once.
    // This is just to make `sync_channel` happy, and wouldn't be present in
    // a real executor.
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}
```

我们在生存者上添加一个方法，使得生成一个新的`future`更加简单。这个方法将会获取一个`future`类型，封装它，然后在内部创建一个新的`Arc<Task>`，它能被放到队列中。

```rust
impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("too many tasks queued");
    }
}
```

为了`poll` `future`，我们需要创建一个`Waker`。就像我们在上一章讨论的一样，`Waker`是负责一旦`wake`被调用时，调度任务再次`poll`。`Waker`告诉执行器哪个任务准备就绪，允许哪些准备取得进展的`future` `poll`。创建一个新的`Waker`最简单的方法是实现`ArcWake` trait，然后使用`wake_ref`或者`.into_waker()` 函数将`Arc<impl ArcWake>`转换为`Waker`。让我们为我们的任务实现`ArcWake`，允许它们转换为`Waker`和唤醒它：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // Implement `wake` by sending this task back onto the task channel
        // so that it will be polled again by the executor.
        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("too many tasks queued");
    }
}
```

当一个`Waker`从`Arc<Task>`中创建，调用`wake()`将会造成一个`Arc`的复制，然后被发送到通道中。我们的执行器需要挑选任务`poll`它。让我们来实现这个：

```rust
impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            // Take the future, and if it has not yet completed (is still Some),
            // poll it in an attempt to complete it.
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                // Create a `LocalWaker` from the task itself
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&*waker);
                // `BoxFuture<T>` is a type alias for
                // `Pin<Box<dyn Future<Output = T> + Send + 'static>>`.
                // We can get a `Pin<&mut dyn Future + Send + 'static>`
                // from it by calling the `Pin::as_mut` method.
                if let Poll::Pending = future.as_mut().poll(context) {
                    // We're not done processing the future, so put it
                    // back in its task to be run again in the future.
                    *future_slot = Some(future);
                }
            }
        }
    }
}
```

我们现在就有一个`future`执行器了。我们甚至可以使用它运行`async/await`代码和自定义的`future`：

```rust
fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    // Spawn a task to print before and after waiting on a timer.
    spawner.spawn(async {
        println!("howdy!");
        // Wait for our timer future to complete after two seconds.
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("done!");
    });

    // Drop the spawner so that our executor knows it is finished and won't
    // receive more incoming tasks to run.
    drop(spawner);

    // Run the executor until the task queue is empty.
    // This will print "howdy!", pause, and then print "done!".
    executor.run();
}
```


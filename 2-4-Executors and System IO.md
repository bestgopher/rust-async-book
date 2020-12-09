在前面关于`Future`的章节中，我们讨论了执行异步操作从套接字中读取数据：

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
            //
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

这个`Future`将从套接字中读取可用的数据，如果没有数据可用，它将会让出给执行器，套接字在次可读的时候任务将被唤醒。然而，从次示例中尚不清楚套接字如何实现的，特别是`set_readable_callback`函数的工作方式尚不清楚。一旦套接字可读，我们怎样安排`wake()`调用呢？一个选择是有一个线程不断检查套接字是否可读，并在适当时调用`wake()`。然而，这是相当低效的，每个阻塞的`future`都需要一个单独的线程。这会大大降低我们异步代码的效率。

实际上，这个问题通过整合IO感知系统阻塞原语来解决的，例如linux上的`epoll`，FreeBSD与Mac OS上的`kqueue`，windows上的`IOCP`，Fuchsia上的`port`(所有都是通过跨平台的rust ctate [`mio`](https://github.com/tokio-rs/mio)暴露出来的)。这些原语都允许一个线程阻塞多个异步IO事件，一旦其中一个事件完成就返回。实际中，这些APIs通常像这样：

```rust
struct IoBlocker {
    /* ... */
}

struct Event {
    // An ID uniquely identifying the event that occurred and was listened for.
    id: usize,

    // A set of signals to wait for, or which occurred.
    signals: Signals,
}

impl IoBlocker {
    /// Create a new collection of asynchronous IO events to block on.
    fn new() -> Self { /* ... */ }

    /// Express an interest in a particular IO event.
    fn add_io_event_interest(
        &self,

        /// The object on which the event will occur
        io_object: &IoObject,

        /// A set of signals that may appear on the `io_object` for
        /// which an event should be triggered, paired with
        /// an ID to give to events that result from this interest.
        event: Event,
    ) { /* ... */ }

    /// Block until one of the events occurs.
    fn block(&self) -> Event { /* ... */ }
}

let mut io_blocker = IoBlocker::new();
io_blocker.add_io_event_interest(
    &socket_1,
    Event { id: 1, signals: READABLE },
);
io_blocker.add_io_event_interest(
    &socket_2,
    Event { id: 2, signals: READABLE | WRITABLE },
);
let event = io_blocker.block();

// prints e.g. "Socket 1 is now READABLE" if socket one became readable.
println!("Socket {:?} is now {:?}", event.id, event.signals);
```

`Futures`执行器可以使用这些原语来提供例如套接字这样的异步IO对象，这些对象可以配置回调函数，当一个特定的IO事件发生时，调用这些回调函数。在上面`SocketRead`例子中的情况下，`socket::set_readable_callback`函数可能看起来像下面的伪代码这样：

```rust
impl Socket {
    fn set_readable_callback(&self, waker: Waker) {
        // `local_executor` is a reference to the local executor.
        // this could be provided at creation of the socket, but in practice
        // many executor implementations pass it down through thread local
        // storage for convenience.
        let local_executor = self.local_executor;

        // Unique ID for this IO object.
        let id = self.id;

        // Store the local waker in the executor's map so that it can be called
        // once the IO event arrives.
        local_executor.event_map.insert(id, waker);
        local_executor.add_io_event_interest(
            &self.socket_file_descriptor,
            Event { id, signals: READABLE },
        );
    }
}
```

现在我们知道仅一个执行器线程就可以接收和发送任何IO事件给合适的`Waker`，它将会唤醒相应的任务，允许执行器驱动更多的任务完成，然后再检查更多的IO事件(如此循环下去)。


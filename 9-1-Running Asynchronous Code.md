一个HTTP服务器应该能够并发地处理多个请求；这就意味着，它不应该处理当前请求之前等待先前的请求完成。The book通过创建一个线程池，用一个线程来处理每个链接来解决这个问题。在这里，我们通过使用异步代码代替通过增加线程来提高吞吐量，实现同样的高效。

我们修改`handle_connection`，返回一个`future`：

```rust
async fn handle_connection(mut stream: TcpStream) {
    //<-- snip -->
}
```

在函数签名时添加`async`，改变函数的返回值类型，使其从`()`改变为实现了`Future<Output=()>` trait的类型。

如果我们尝试编译这个，编译器将会警告我们这将不能工作：

```shell
$ cargo check
    Checking async-rust v0.1.0 (file:///projects/async-rust)
warning: unused implementer of `std::future::Future` that must be used
  --> src/main.rs:12:9
   |
12 |         handle_connection(stream);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(unused_must_use)]` on by default
   = note: futures do nothing unless you `.await` or poll them
```

因为我们没有`.await`或者`poll` `handle_connection`返回的结果，它将不能运行。如果你运行服务器，然后在浏览器中访问`127.0.0.1:7878`，你将会看到链接被拒绝；我们的服务并没有处理请求。

我们不能在同步代码中`.await`或者`poll` `future`。我们需要一个异步运行时来处理调度和运行`future`完成。请看前面章节获取异步运行时，`executor`和`reactors`的更多信息。



### 添加异步运行时

这里，我们使用一个`async-std` crate中的执行器。`async-std`中的`#[async_std::main]`属性允许我们编写一个异步main函数。为了使用这个，在`Cargo.toml`中启用`async-std`的`attributes`特性。

```toml
[dependencies.async-std]
version = "1.6"
features = ["attributes"]
```

第一步，我们切换到一个异步的man函数，然后在`handle_connection`的异步版本返回的`future`上执行`.await`操作。接下来，我们测试服务怎样响应的。这里看起来就像：

```rust
#[async_std::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        // Warning: This is not concurrent!
        handle_connection(stream).await;
    }
}
```

现在，让我们测试服务是否能并发地处理链接。简单地让`handle_connection` 异步并不意味着服务能够同时处理多个请求，接下来我们将会知道为什么会这样。

为了说明这个，让我们模拟一个慢请求。当一个客户端请求`127.0.0.1:7878/sleep`时，我们的服务将会睡眠5秒钟：

```rust
use async_std::task;

async fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else if buffer.starts_with(sleep) {
        task::sleep(Duration::from_secs(5)).await;
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };
    let contents = fs::read_to_string(filename).unwrap();

    let response = format!("{}{}", status_line, contents);
    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

这与the Book中模拟慢请求非常类似，但是最重要的不同是：我们使用的是非阻塞函数`async_std::task::sleep`替代阻塞函数`std::thread::sleep`。记住即使代码是通过`async fn`和`await`运行的，它也可能是阻塞的。要测试我们的服务处理链接是否是并发的，我们需要确认`handle_connection`是非阻塞的。

如果你启动了服务，你将会看到请求`127.0.0.1:7878/sleep`将会阻塞其他进来的请求5秒钟！这是因为当我们对`handle_connection`进行`.await`操作时，没有其他并发的任务能取得进展。在下面的章节中，我们将会看到怎样使用异步代码并发地处理链接。


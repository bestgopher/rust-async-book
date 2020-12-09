我们继续测试我们的`handle_connection`函数。

首先，我们需要一个`TcpStream`来使用。在一个端对端或者集成测试中，我们可能想要创建一个TCP链接来测试我们的代码。一个方法是监听本地的端口0。端口0不是一个无效的UNIX端口，它用作测试。操作系统将会选择一个可用的TCP端口给我们。

在这个例子中，我们将会为链接处理器编写一个单元测试，检查每个输出对应的输出是否正确。为了保持单元测试的隔离性和正确性，我们模拟一个`TcpStream`。

首先，我们改变`handle_connection`的签名让它更加容易测试。`handle_connection`实际上并不需要`async_std::net::TcpStream`；它需要一个实现了`async_std::io::Read`和`async_std::io::Write` 和`marker::Unpin`的任何结构体。修改参数签名以便我们可以传递一个mock用于测试。

```rust
use std::marker::Unpin;
use async_std::io::{Read, Write};

async fn handle_connection(mut stream: impl Read + Write + Unpin) {
```

接下来，我们mock一个实现了这些trait的`TcpStream`。首先，我们实现`Read` trait的`poll_read`方法。我们mock的`TcpStream`将包含一些被拷贝进read buffer的数据，我们将会返回`Poll::Ready`，以此表示读取完毕。

```rust
use super::*;
use futures::io::Error;
use futures::task::{Context, Poll};

use std::cmp::min;
use std::pin::Pin;

struct MockTcpStream {
  read_data: Vec<u8>,
  write_data: Vec<u8>,
}

impl Read for MockTcpStream {
  fn poll_read(
    self: Pin<&mut Self>,
    _: &mut Context,
    buf: &mut [u8],
  ) -> Poll<Result<usize, Error>> {
    let size: usize = min(self.read_data.len(), buf.len());
    buf[..size].copy_from_slice(&self.read_data[..size]);
    Poll::Ready(Ok(size))
  }
}
```

我们实现的`Write`非常类似，虽然我们需要写三个方法：`poll_write`，`poll_flush`，`poll_close`。`poll_write`将会复制传入的数据到mock的`TcpStream`，当完成时返回`Poll::Ready`。不需要为`TcpStream`实现`flush`或者`close`，因此在`poll_flush`和`poll_close`直接返回`Poll::Ready`。

```rust
impl Write for MockTcpStream {
	fn poll_write(
    mut self: Pin<&mut Self>,
    _: &mut Context,
    buf: &[u8],
  ) -> Poll<Result<usize, Error>> {
    self.write_data = Vec::from(buf);
    return Poll::Ready(Ok(buf.len()));
  }
  
  fn poll_flush(self: Pin<&mut Self>, _: &mut Context) -> Poll<Result<(), Error>> {
    Poll::Ready(Ok(()))
  }
  
  fn poll_close(self: Pin<&mut Self>, _: &mut Context) -> Poll<Result<(), Error>> {
    Poll::Ready(Ok(()))
  }
}
```

最后，我们的mock需要实现`Unpin`，表示它在内存的位置可以安全地move。更多关于`Pin`和`Unpin`的信息，参考前面章节。

```rust
use std::marker::Unpin;

impl Unpin for MockTcpStream {}
```

现在，我们已经准备测试`handle_connection`函数。设置包含一些初始化信息的`MockTcpStream`之后，我们可以通过属性`#[async_std::test]`来运行`handle_connection`，这与我们使用`#[async_std::main]`类似。为了确保`handle_connection`如预期一样运行，我们将检查基于它初始的文本是否写入了`MockTcpStream`。

```rust
use std::fs;

#[async_std::test]
async fn test_handle_connection() {
  let input_bytes = b"GET / HTTP/1.1\r\n";
  let mut contents = vec![0u8; 1024];
  contents[..input_bytes.len()].clone_from_slice(input_bytes);
  let mut stream = MockTcpStream {
    read_data: contents,
    write_data: Vec::new(),
  };

  handle_connection(&mut stream).await;
  let mut buf = [0u8; 1024];
  stream.read(&mut buf).await.unwrap();

  let expected_contents = fs::read_to_string("hello.html").unwrap();
  let expected_response = format!("HTTP/1.1 200 OK\r\n\r\n{}", expected_contents);
  assert!(stream.write_data.starts_with(expected_response.as_bytes()));
}
```


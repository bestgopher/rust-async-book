###  Stream trait

`Stream` trait 与 `Future` trait类型，但是可以在完成之前让出多个值，与标准库的`Iterator` trait类似。

```rust
trait Stream {
    /// The type of the value yielded by the stream.
    type Item;

    /// Attempt to resolve the next item in the stream.
    /// Returns `Poll::Pending` if not ready, `Poll::Ready(Some(x))` if a value
    /// is ready, and `Poll::Ready(None)` if the stream has completed.
  	/// 尝试解析流中的下一项。
  	/// 如果没有准备好，返回`Poll::Pending`，准备好了返回`Poll::Ready(Some(x))`
    /// 流完成了返回`Poll::Ready(None)`.
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}
```

一个`Stream`常见的例子是`futures` crate中的管道类型的`Receiver`。当`Sender`端每次发送一个数据，它将会抛出`Some(val)`，一旦`Sender`被drop且所有待定的信息已经到达了，将会抛出`None`。

```rust
async fn send_recv() {
    const BUFFER_SIZE: usize = 10;
    let (mut tx, mut rx) = mpsc::channel::<i32>(BUFFER_SIZE);

    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    drop(tx);

    // `StreamExt::next` is similar to `Iterator::next`, but returns a
    // type that implements `Future<Output = Option<T>>`.
    assert_eq!(Some(1), rx.next().await);
    assert_eq!(Some(2), rx.next().await);
    assert_eq!(None, rx.next().await);
}
```




`futures::join`宏并发地执行不同的futures，等待其完成。

#### join!

当执行多个异步操作，一系列简单的`.await`是很诱人的。

```rust
async fn get_book_and_music() -> (Book, Music) {
    let book = get_book().await;
    let music = get_music().await;
    (book, music)
}
```

然而，这会很慢，因为它不会尝试在`get_book`完成前`get_music`。在一些其他语言，`futures`几乎可以完全运行，因此两个操作可以通过首先调用每个`async fn`开始`futures`来并发运行，然后两个同时等待：

```rust
// WRONG -- don't do this
async fn get_book_and_music() -> (Book, Music) {
    let book_future = get_book();
    let music_future = get_music();
    (book_future.await, music_future.await)
}
```

然而，Rust的`futures`在`await`之前不会做任何工作。这就意味着上面的两个代码片段串行地执行`book_future`和`music_future`而不是并行。为了正确地并行运行两个`futures`，使用`futures::join!`：

```rust
use futures::join;

async fn get_book_and_music() -> (Book, Music) {
    let book_fut = get_book();
    let music_fut = get_music();
    join!(book_fut, music_fut)
}
```

`join!`返回的是一个包含每个`Future`输出的元组。



### try_join!

`futures`为了返回`Result`，考虑使用`try_join!`而不是`join!`。因为`join!`只有所有子`future`完成了才算完成，即使子`futures`返回一个`Err`，它也会继续执行其他子`future`。

与`join!`不同，如果有一个子`future`完成，`try_join!`会立马完成。

```rust
use futures::try_join;

async fn get_book() -> Result<Book, String> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book();
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

记住传递进`try_join!`的`future`必须有相同的error类型。考虑使用`futures::future::TryFutureExt`中的`.map_err(|e| ...)` 和`.err_into`函数来统一error类型。

```rust
use futures::{
    future::TryFutureExt,
    try_join,
};

async fn get_book() -> Result<Book, ()> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book().map_err(|()| "Unable to get book".to_string());
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```




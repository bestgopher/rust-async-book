与同步的`Iterator`相似，有许多不同的方法迭代和处理`Stream`中的值。有组合风格的方法，例如`map`，`filter` 和 `fold`，以及它们遇到错误就退出的版本`try_map`，`try_filter`，`try_fold`。

不幸的是，`Stream`不能使用`for`循环，但是对于函数式编程风格的代码，`while let` 和 `next/try_next`函数可以使用：

```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::StreamExt; // for `next`
    let mut sum = 0;
    while let Some(item) = stream.next().await {
        sum += item;
    }
    sum
}

async fn sum_with_try_next(
    mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>,
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt; // for `try_next`
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? {
        sum += item;
    }
    Ok(sum)
}
```

然而，如果我们同一时刻只处理一个元素，我们可能会留下并发的机会，毕竟，这就是我们为什么要首先编写异步代码的原因。为了并发地处理`Stream`中的多个Item，使用`for_each_concurrent` 和 `try_for_each_concurrent` 方法。

```rust
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>,
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt; // for `try_for_each_concurrent`
    const MAX_CONCURRENT_JUMPERS: usize = 100;

    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;

    Ok(())
}
```


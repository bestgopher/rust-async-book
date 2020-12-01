`futures::select`宏同时运行多个futures，允许用户在任何`future`完成时做出响应。

```rust
use futures::{
    future::FutureExt, // for `.fuse()`
    pin_mut,
    select,
};

async fn task_one() { /* ... */ }
async fn task_two() { /* ... */ }

async fn race_tasks() {
    let t1 = task_one().fuse();
    let t2 = task_two().fuse();

    pin_mut!(t1, t2);

    select! {
        () = t1 => println!("task one completed first"),
        () = t2 => println!("task two completed first"),
    }
}
```

这个函数将会并发地允许`t1`和`t2`。当`t1`或者`t2`完成了，相应的处理器(handler)将会调用`println!`，函数将会在剩余标记的任务还未完成的情况下结束。

`select`的基础语法是`<pattern> = <expression> => <code>，`，重复你想要`select`的`future`。



### default => ... 和 complete => ...

`select`也支持`default`和`complete`分支。

没有`future`完成时，`default`将会被执行。`select`的`default`分支将会立马返回，因为没有其他`futures`准备好了。

`complete`分支可用于所有被`select`的`futures`都已经完成。这通常处理循环一个`select!`。

```rust
use futures::{future, select};

async fn count() {
    let mut a_fut = future::ready(4);
    let mut b_fut = future::ready(6);
    let mut total = 0;

    loop {
        select! {
            a = a_fut => total += a,
            b = b_fut => total += b,
            complete => break,
            default => unreachable!(), // never runs (futures are ready, then complete)
        };
    }
    assert_eq!(total, 10);
}
```



### 与 `Upin` 和 `FusedFuture`交互

在上面第一个例子中你可能注意到我们在两个`async fn`返回的`futures`上调用了`.fuse()`，以及通过`pin_mut`固定了它们。这些都是必要的，因为在`select`中使用的`future`必须实现`Unpin`和`FusedFuture`特征。

`Unpin`是必要的，因为`select`中使用的future不是获取的值，而是可变引用。因为不获取`future`的所有权，没完成的`futures`可以在调用`select`时再次使用。

类似地，`FusedFuture`也是必要的，因为`select`不能轮询一个已完成的`future`。实现了`FusedFuture`的`future`会追踪它们是否完成。这让`select`在循环中使用变为可能，只轮询未完成的`future`。看看上面的例子，`a_fut`或者`b_fut`将在第二次循环的时候完成。因为通过`future::ready`返回的`future`实现了`fusedFuture`，它将会告诉`select`不要再次轮询了。

记住streams也有相应的`FusedStream`特征。实现了这个特征或者通过`.fuse()`包装的Streams将会通过它们的`.next()`或者`.try_next()`组合器生成`FusedFuture`的`future`。

```rust
use futures::{
    stream::{Stream, StreamExt, FusedStream},
    select,
};

async fn add_two_streams(
    mut s1: impl Stream<Item = u8> + FusedStream + Unpin,
    mut s2: impl Stream<Item = u8> + FusedStream + Unpin,
) -> u8 {
    let mut total = 0;

    loop {
        let item = select! {
            x = s1.next() => x,
            x = s2.next() => x,
            complete => break,
        };
        if let Some(next_num) = item {
            total += next_num;
        }
    }

    total
}
```



### 通过`Fuse`和`FuturesUnordered`在循环中`select`实现并发任务

一个有些难以发现，但是很方便的函数是`Fuse::terminated()`，它可以构造一个空的且终止的`future`，并且随后可以被一个需要运行的`future`填充。

当有一个任务需要在`select`的循环中运行，且此任务在循环的内部创建时非常方便。

记住`.select_next_some()`函数的作用。它可以与`select`配合，只运行从流中返回的值为`Some(_)`的分支，

忽略`None`。

```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) { /* ... */ }

async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let run_on_new_num_fut = run_on_new_num(starting_num).fuse();
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(run_on_new_num_fut, get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // The timer has elapsed. Start a new `get_new_num_fut`
                // if one was not already running.
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // A new number has arrived-- start a new `run_on_new_num_fut`,
                // dropping the old one.
                run_on_new_num_fut.set(run_on_new_num(new_num).fuse());
            },
            // Run the `run_on_new_num_fut`
            () = run_on_new_num_fut => {},
            // panic if everything completed, since the `interval_timer` should
            // keep yielding values indefinitely.
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```

当有许多相同`future`的副本需要同时运行时，使用`FuturesUnordered`类型。下面的例子与上面的类似，但是会运行每个`run_on_new_num_fut`的副本直到完成，而不是在创建新的时中止它们。它也会打印`run_on_new_num_fut`的返回值。

```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, FuturesUnordered, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) -> u8 { /* ... */ 5 }

// Runs `run_on_new_num` with the latest number
// retrieved from `get_new_num`.
//
// `get_new_num` is re-run every time a timer elapses,
// immediately cancelling the currently running
// `run_on_new_num` and replacing it with the newly
// returned value.
async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let mut run_on_new_num_futs = FuturesUnordered::new();
    run_on_new_num_futs.push(run_on_new_num(starting_num));
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // The timer has elapsed. Start a new `get_new_num_fut`
                // if one was not already running.
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // A new number has arrived-- start a new `run_on_new_num_fut`.
                run_on_new_num_futs.push(run_on_new_num(new_num));
            },
            // Run the `run_on_new_num_futs` and check if any have completed
            res = run_on_new_num_futs.select_next_some() => {
                println!("run_on_new_num_fut returned {:?}", res);
            },
            // panic if everything completed, since the `interval_timer` should
            // keep yielding values indefinitely.
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```


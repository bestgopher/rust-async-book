在内部，`async fn`创建一个包含被`.await`的子`Future`的状态机。它使得递归的`async fn`有点棘手，因为结果状态机类型必须包含自身：

```rust
// This function:
async fn foo() {
    step_one().await;
    step_two().await;
}
// generates a type like this:
enum Foo {
    First(StepOne),
    Second(StepTwo),
}

// So this function:
async fn recursive() {
    recursive().await;
    recursive().await;
}

// generates a type like this:
enum Recursive {
    First(Recursive),
    Second(Recursive),
}
```

它将不能工作 - 我们创建一个无限大小的类型！编译器将会报错：

```rust
error[E0733]: recursion in an `async fn` requires boxing
 --> src/lib.rs:1:22
  |
1 | async fn recursive() {
  |                      ^ an `async fn` cannot invoke itself directly
  |
  = note: a recursive `async fn` must be rewritten to return a boxed future.
```

为了达到目的，我们不得不使用`Box`引入间接的引用。不幸的是，编译器的限制意味着仅将对`recursive()`的调用包装在`Box :: pin`中是不够的。为了使它工作，我们不得不使递归进一个非`async`的函数中，这个函数返回一个`.boxed() async`块。

```rust
use futures::future::{BoxFuture, FutureExt};

fn recursive() -> BoxFuture<'static, ()> {
    async move {
        recursive().await;
        recursive().await;
    }.boxed()
}
```


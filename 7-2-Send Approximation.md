有些`async fn`状态机可以安全的在线程间传递，而有些则不能。`async fn`是否是`Send`取决于`.await`是否包含非`Send`类型。编译器尽其所能来评估值在跨`await`点时值是否可能会保留，但是这种分析在今天的很多地方都太保守了。

例如，考虑一个简单的非`Send`的类型-一个包含`Rc`的类型

```rust
use std::rc::Rc;

#[derive(Default)]
struct NotSend(Rc<()>);
```

`NotSend`类型的变量可以在`async fn`中短暂地视作临时变量，即使`async fn`返回的`Future`类型必须为`Send`。

```rust
async fn bar() {}
async fn foo() {
    NotSend::default();
    bar().await;
}

fn require_send(_: impl Send) {}

fn main() {
    require_send(foo());
}
```

然而， 如果我们改变`foo`来保存`NotSend`到一个变量中，下面的例子将不能编译：

```rust
async fn foo() {
    let x = NotSend::default();
    bar().await;
}
```

```shell
error[E0277]: `std::rc::Rc<()>` cannot be sent between threads safely
  --> src/main.rs:15:5
   |
15 |     require_send(foo());
   |     ^^^^^^^^^^^^ `std::rc::Rc<()>` cannot be sent between threads safely
   |
   = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<()>`
   = note: required because it appears within the type `NotSend`
   = note: required because it appears within the type `{NotSend, impl std::future::Future, ()}`
   = note: required because it appears within the type `[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]`
   = note: required because it appears within the type `std::future::GenFuture<[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]>`
   = note: required because it appears within the type `impl std::future::Future`
   = note: required because it appears within the type `impl std::future::Future`
note: required by `require_send`
  --> src/main.rs:12:1
   |
12 | fn require_send(_: impl Send) {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
```

这个错误是正确的。如果我们存储的`x`到一个变量中，它将直到`.await`后才会被drop，在这里`async fn`可能会在不同的线程上运行。因为`Rc`不是`Send`的，所以不允许在不同的线程中传递。一个简单的解决方式是在`.await`之前drop掉`Rc`，但是不幸的是，今天这个方法不能工作。

为了成功解决这个问题，你可以引入一个作用域来分装任何非`Send`的变量。这将非常简单地告诉编译器这些变量不能存活到`.await`点。

```rust
async fn foo() {
    {
        let x = NotSend::default();
    }
    bar().await;
}
```


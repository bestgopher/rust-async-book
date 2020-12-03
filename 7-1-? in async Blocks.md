就像`async fn`中一样，在`async`块中也一样能够使用`?`。然而，`async`块返回的类型不能明确说明的。这可能造成编译器不能推断`async`块的error类型。

例如，下面代码：

```rust
let fut = async {
    foo().await?;
    bar().await?;
    Ok(())
};
```

将会触发这个错误：

```shell
error[E0282]: type annotations needed
 --> src/main.rs:5:9
  |
4 |     let fut = async {
  |         --- consider giving `fut` a type
5 |         foo().await?;
  |         ^^^^^^^^^^^^ cannot infer type
```

不幸的是，当前无法给`fut`一个类型，也没有明确指定`async`块返回值类型的方式。为了解决这个问题，使用“ turbofish”运算符为异步块提供成功和错误类型：

```rust
let fut = async {
    foo().await?;
    bar().await?;
    Ok::<(), MyError>(()) // <- note the explicit type annotation here
};
```


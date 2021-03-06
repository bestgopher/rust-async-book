### Pinning

为了轮询 futures，它们必须被一个称为`Pin<T>`的特殊类型固定。如果你已经阅读了以前的章节-`Executing Futures and Tasks`的`the Future trait`小节，你将在`Future::poll`方法中定义的`self::Pin<&mut Self>`认识到`Pin`。但是它是什么意思，我们为什么需要它？



### Why Pinning

`Pin`与`Unpin`标记协同工作。固定保证实现了`!Unpin`的对象永远不会被move。为了理解为什么它是必要的，我们需要记住`async`/`.await`是怎么工作的。考虑以下代码：

```rust
let fut_one = /* ... */;
let fut_two = /* ... */;
async move {
    fut_one.await;
    fut_two.await;
}
```

在底层，这个创建了一个匿名类型，它实现了`Future` trait，提供了一个`poll`方法，就像这样：

```rust
// The `Future` type generated by our `async { ... }` block
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// List of states our `async` block can be in
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

当`poll`第一次被调用，它将轮训`fut_one`。如果`fut_one`还未完成，`AsyncFuture::poll`将会返回。Future 对`poll`进行调用，将在上一个停止的地方继续。这个过程持续到futures能够成功完成为止。

然而，如果我有一个使用了引用的`async`块，将会发生什么呢？例如：

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

这个编译成什么结构呢？

```rust
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // points to `x` below
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}
```

这里，`ReadIntoBuf` future从我们结构体的字段`x`获取了引用。然而，如果`AsyncFuture` 被 move 了，本地的`x`

也将会move，使存储在`read_into_buf_fut.buf`的指针失效。

把futures固定在内存的一个特定的地方来防止这个问题，使在一个`async`块中创建一个引用变得安全。



### Pinning in detail

让我们通过使用一个稍微简单的例子来尝试理解固定。以上我们遇到的问题最终归结为我们怎么在自指类型(self-referential types)中处理引用。

现在我们的例子像这样：

```rust
use std::pin::Pin;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

`Test`提供了方法来获取字段`a`和`b`的引用。因为`b`是一个`a`的引用，且rust的借用规则不允许我们定义它的生命周期，因此我们将它存储为指针。我们现在有一个自指的结构体了。

如果我们没有移动任何数据，示例将运行良好，如通过运行下面的示例可以观察到的：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

我们获得了我们期待的结果：

```shell
a: test1, b: test1
a: test2, b: test2
```

如果我们通过交换`test1`和`test2`，从而move数据，会发生什么呢：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());
}
```

天真地，我们可能会认为将会活动`test1`的调试打印2次：

```rust
a: test1, b: test1
a: test1, b: test1
```

但是实际上：

```shell
a: test1, b: test1
a: test1, b: test2
```

`test2.b`的指针始终指向`test1`内部的旧位置。这个结构体不再是自指的了，它持有不同对象的字段的指针。这意味着我们不能依靠`test2.b`的生命周期来帮的`test2`的生命周期。

如果你还没理解，下面的例子将最终让你理解：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

下面的图形将发生的一切形象化：

![aaa](./images/swap_problem.jpg)

很容易使它显示未定义的行为并以其他引人注目的方式失败。



### Pinning in Practice

让我们看看`Pin`类型怎么帮助我们解决固定的问题。

`Pin`类型包装了指针类型，保证指针背后的值不会被移动。例如：`Pin<&mut T>`、`Pin<&T>`、`Pin<Box<T>>`

都能保证如果 `T:!Unpin`，`T`将不会被move。

多数类型在被移动时都没问题。这些类型实现了一个叫做`Unpin`的trait。`Unpin`类型的指针能自由放置到`Pin`或从`Pin`中取出。例如，`u8`是`Unpin`，因此`Pin<&mut u8>`的行为和普通的`&mut u8`一样。

然而，被一个`!Unpin`固定的类型不能被移动。通过`async/await`创建的Futures就是这样的一个例子。



### Pinning to the Stack

回到我们的例子。我们可以通过使用`Pin`解决我们的问题。让我们看看例子通过固定指针替换了是什么样的。

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, // This makes our type `!Unpin`
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

如果我们的类型实现了`!Unpin`，将对象固定到栈上将总是`unsafe`的。当你固定对象在栈上是，你可以使用像`pin_utils`这样的crate来避免写`unsafe`的代码。

下面，我们固定对象`test1`和`test2`到栈上。

```rust
pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from being accessed again
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

现在，如果我们尝试移动我们的数据，我们将会得到一个编译错误。

```rust
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

类型系统阻止了我们移动数据。

记住栈固定总是依赖你编写的`unsafe`代码提供的保证。

虽然我们知道`＆'a mut T`的指针在`'a`的生命周期中固定，但我们不知道`'a`结束后数据`＆'a mut T`指向的数据是否没有移动。 如果这样做，将违反`Pin`的约定。

容易犯的一个错误就是忘记隐藏原始变量，因为您可以drop `Pin`并将数据移动到`＆'a mut T`之后，如下所示（这违反了`Pin`约定）：

```rust
fn main() {
   let mut test1 = Test::new("test1");
   let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
   Test::init(test1_pin.as_mut());
   drop(test1_pin);
   println!(r#"test1.b points to "test1": {:?}..."#, test1.b);
   let mut test2 = Test::new("test2");
   mem::swap(&mut test1, &mut test2);
   println!("... and now it points nowhere: {:?}", test1.b);
}
```



### Pinning to the Heap

固定一个`!Unpin`类型到堆上，给予我们的数据一个稳定的地址，因此我们知道指向的数据在它被固定后不能被移动。在栈固定的约定中，我们知道知道对象在生命周期内将会被固定。

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

一个函数需要`Unpin`的futures。为了在需要一个`Unpin`类型的函数中使用不是`Unpin`的`Future`或者`Stream`，你可以使用`Box::pin`(创建一个`Pin<Box<T>>`)，或者`pin_utils::pin_mut!`宏(创建一个`Pin<&mut T>`)来获取一个固定的值。`Pin<Box<Fut>>`和`Pin<&mut Fut>`都可以作为Future使用，也都实现了`Unpin`。

例如：

```rust
use pin_utils::pin_mut; // `pin_utils` is a handy crate available on crates.io

// A function which takes a `Future` that implements `Unpin`.
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { /* ... */ }

let fut = async { /* ... */ };
execute_unpin_future(fut); // Error: `fut` does not implement `Unpin` trait

// Pinning with `Box`:
let fut = async { /* ... */ };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK

// Pinning with `pin_mut!`:
let fut = async { /* ... */ };
pin_mut!(fut);
execute_unpin_future(fut); // OK
```



### 概要

1. 如果`T: Unpin`(这是默认的)，`Pin<'a, T>` 完全与`&'a mut T` 相等。换句话说：`Unpin`意味着即使此类型被固定，它也能被移动，因此`Pin`在此类型上没有作用。
2. 如果`T: !Unpin`，将`&mut T`固定为`T` 需要`unsafe`。
3. 很斗殴标准库的类型实现了`Unpin`。你遇到的大多数常规类型也是如此。`async/await`产生的`Future`是一个例外。
4. 在 nightly版本中，通过一个特性标志(feature flag)，你可以在一个类型上添加一个`!Unpin` bound，或者在稳定版中，添加`std::marker::PhantomPinned`到你的类型上。
5. 你可以将数据固定在栈或者堆上。
6. 固定一个`!Unpin`对象到栈上需要`unsafe`。
7. 固定一个`!Unpin`对象到堆上不需要`unsafe`。使用`Box::pin`能做到这个。
8. 对于固定`T: !Unpin`的数据，你必须保持不变，即从它被固定到调用`drop`这段时间中，内存不会失效或从新定位。这对于`Pin`约定是很重要的部分。


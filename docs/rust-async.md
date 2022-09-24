# Rust 异步编程笔记

## Future trait

Rust 异步编程最核心的是 [Future](https://doc.rust-lang.org/std/future/trait.Future.html) trait：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Future` 代表一个异步计算，它会产生一个值。通过调用 `poll` 方法来推进 `Future` 的运行，如果 `Future` 完成了，它将返回 `Poll::Ready(result)`，我们拿到运算结果。如果 `Future` 还不能完成，可能是因为需要等待其他资源，它返回 `Poll::Pending`。等条件具备，如资源已经准备好，这个 `Future` 将被唤醒，再次进入 `poll`，直到计算完成获得结果。

## async/.await

如果产生一个 `Future` 呢，使用 `async` 是产生 `Future` 最方便的方法。使用 `async` 有两种方式：`async fn` 和 `async blocks`。每种方法都返回一个实现了`Future` trait 的匿名结构：

```rust
// `foo()` returns a type that implements `Future<Output = u8>`.
async fn foo() -> u8 { 5 }

fn bar() -> impl Future<Output = u8> {
    // This `async` block results in a type that implements
    // `Future<Output = u8>`.
    async {
        5
    }
}
```

这两种方式是等价的，都返回了 `impl Future<Output = u8>`。`async` 关键字相当于一个返回 `impl Future<Output>` 的语法糖。

调用 `async fn` 并不会让函数执行，而是返回 `impl Future<Output>`，你只有在返回值上使用 `.await`，才能触发函数的实际执行。

```rust
async fn say_world() {
    println!("world");
}

#[tokio::main]
async fn main() {
    // Calling `say_world()` does not execute the body of `say_world()`.
    let op = say_world();

    // This println! comes first
    println!("hello");

    // Calling `.await` on `op` starts executing `say_world`.
    op.await;
}
```

上面的程序输出为：

```shell
hello
world
```

在 `Future` 上调用 `await`，相当于执行 `Future::poll`。如果 `Future` 被某些条件阻塞，它将放弃对当前线程的控制。当条件准备好后， `Future`会被唤醒恢复执行。

简单总结，我们用`async` 生成 `Future`，用 `await` 来触发 `Future` 的执行。尽管其他语言也实现了async/.await，但 Rust 的 async 是 lazy 的，只有在主动 await 后才开始执行。

我们当然也可以手工为数据结构实现 `Future`：

```rust
struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

同样用 `await` 触发 Future 的实际执行：

```rust
#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

## Pin

`Future` 被每个 `await` 分成多段，执行到 `await` 可能因为资源没准备好而让出 CPU 暂停执行，随后该 `future` 可能被调度到其他线程接着执行。所以 `future` 结构中需要保存跨await的数据，形成了自引用结构。

<img src="https://cdn.mazhen.tech/images/202209201453379.jpg" alt="future" style="zoom:50%;" />

自引用结构不能被移动，否则内部引用因为指向移动前的地址，引发不可控的问题。所以future需要被pin住，不能移动。

<img src="https://cdn.mazhen.tech/images/202209201529136.webp" alt="Self-Referential Structure" style="zoom: 33%;" />

如何让 `future` 不被move？ 方法调用时只传递引用，那么就没有移动 `future`。但是通过可变引用仍然可以使用 replace，swap 等方式移动数据。那么用 `Pin` 包装可变引用 `Pin<&mut T>`，让用户没法拿到 `&mut T`，就把这个漏洞堵上了。

<img src="https://cdn.mazhen.tech/images/202209201445723.webp" alt="pin" style="zoom: 33%;" />

总之 `Pin<&mut T>` 不是数据的 owner，也没法获得 `&mut T`，所以就不能移动 T。

注意，`Pin` 拿住的是一个可以解引用成 T 的指针类型 P，而不是直接拿原本的类型 T。Pin 本身是可 move 的，T 被 pin 住，不能 move。

## async runtime的内部实现

要运行异步函数，必须将最外层的 `Future` 提交给 executor。 executor 负责调用Future::poll，推动异步计算的前进。

![executor](https://cdn.mazhen.tech/images/202209201433572.png)

`executor` 内部会有一个 `Task` 队列，`executor` 在 `run` 方法内，不停的从 `receiver` 获取 `Task`，然后执行。

`Task` 包装了一个 `future`，同时内部持有一个 `sender`，用于将自身放回 executor 的 Task 队列。

`Future` 的 `poll` 方法，接收的是 `Pin<&mut Self>`，而不是 `&mut Self`。所以在向 executor 提交 Future 时，需要先 pin 住，然后才能用来初始化 Task：

```rust
fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
where
    F: Future<Output = ()> + Send + 'static,
{
    let task = Arc::new(Task {
        future: Mutex::new(Box::pin(future)),
        executor: sender.clone(),
    });

    let _ = sender.send(task);
}
```

保存在 Task 字段中的 Future 是 `Pin<Box<Future>>`，保证了以后每次调用 `poll` 传入的是 `Pin<&mut Self>`。注意，Pin 是可以移动的，Task 也是可以移动的，只是 Future 不能移动。

在执行 Future 时，如果遇到资源未准备好，需要让出 CPU，那么 Task 可以将自己放入 Reactor。Task 实现了 `ArcWake trait`，实际上放入 Reactor 的 Waker 就是 Task 的包装：

```rust
fn poll(self: Arc<Self>) {
    // Get a waker referencing the task.
    let waker = task::waker(self.clone());
    // Initialize the task context with the waker.
    let mut cx = Context::from_waker(&waker);

    // This will never block as only a single thread ever locks the future.
    let mut future = self.future.try_lock().unwrap();

    // Poll the future
    let _ = future.as_mut().poll(&mut cx);
}
```

当 Reactor 得到了满足条件的事件，它会调用 `Waker.wake()` 唤醒之前挂起的任务。Waker.wake 会调用 `Task::wake_by_ref` 方法，将 Task 放回 executor 的任务队列：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let _ = arc_self.executor.send(arc_self.clone());
    }
}
```

## Stream trait

对于 `Iterator`，可以不断调用其 `next()` 方法，获得新的值，直到 `Iterator` 返回 `None`。`Iterator` 是阻塞式返回数据的，每次调用 `next()`，必然独占 CPU 直到得到一个结果，而异步的 `Stream` 是非阻塞的，在等待的过程中会空出 CPU 做其他事情。

`Stream::poll_next()` 方法和 `Future::poll()` 类似, 除了它可以被重复调用，以便从 `Stream` 中接收多个值。然而，`poll_next()` 调用起来不方便，我们需要自己处理 Poll 状态。也就是说，`await` 语法糖只能应用在 Future 上，没法使用 `stream.await` 。所以，我们要想办法用 `Future` 包装 `Stream`，在 `Future::poll()` 中调用 `Stream::poll_next()`，这样就可以使用 await。 [StreamExt](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html) 提供了 `next()` 方法，返回一个实现了 `Future trait` 的 `Next` 结构，这样，我们就可以直接通过 `stream.next().await` 来获取下一个值了。看一下 `next()` 方法以及 `Next` 结构的实现。

```rust
pub trait StreamExt: Stream {
    fn next(&mut self) -> Next<'_, Self> where Self: Unpin {
        assert_future::<Option<Self::Item>, _>(Next::new(self))
    }
}

// next 返回的 Next 结构
pub struct Next<'a, St: ?Sized> {
    stream: &'a mut St,
}

// Next 实现了 Future，每次 poll() 实际上就是从 stream 中 poll_next()
impl<St: ?Sized + Stream + Unpin> Future for Next<'_, St> {
    type Output = Option<St::Item>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.stream.poll_next_unpin(cx)
    }
}
```

当手动实现一个 stream 时，它通常是通过合成 futures 和其他stream 来完成的。例如下面的例子，将 [Lines](https://docs.rs/tokio/1.14.0/tokio/io/struct.Lines.html) 封装为 Stream，在 Stream::poll_next() 中利用了 Lines::poll_next_line()：

```rust
#[pin_project]
struct LineStream<R> {
    #[pin]
    lines: Lines<BufReader<R>>,
}

impl<R: AsyncRead> LineStream<R> {
    /// 从 BufReader 创建一个 LineStream
    pub fn new(reader: BufReader<R>) -> Self {
        Self {
            lines: reader.lines(),
        }
    }
}

/// 为 LineStream 实现 Stream trait
impl<R: AsyncRead> Stream for LineStream<R> {
    type Item = std::io::Result<String>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        self.project()
            .lines
            .poll_next_line(cx)
            .map(Result::transpose)
    }
}
```

`Stream` 可以是 unpin 的，`Future` 也可以是 unpin 的，如果他们内部包含了其他 `!Unpin` 的 `Stream` 或 `Future`，只需要把他们用 pin 包装，外面的 `Stream` 和 `Future` 就可以是 unpin 的。

一般我们使用的 `Stream` 都是 unpin 的，如果不是，就用 pin 把它变成 unpin 的。为啥我们用的都是 unpin 的？因为能 move 的 Stream 更加灵活，可以作为参数和返回值。

## AsyncRead 和 AsyncWrite

所有同步的 Read / Write / Seek trait，前面加一个 Async，就构成了对应的异步 IO 接口。

AsyncRead / AsyncWrite 的方法会返回一个实现了 `Future` 的 `struct`，这样我们才能使用 await ，将 future 提交到 async runtime，触发 future 的执行。例如 `AsyncReadExt::read_to_end()`方法，返回 `ReadToEnd` 结构，而 `ReadToEnd` 实现了 Future：

```rust
pub trait AsyncReadExt: AsyncRead {
    ...
    fn read_to_end<'a>(&'a mut self, buf: &'a mut Vec<u8>) -> ReadToEnd<'a, Self>
    where
        Self: Unpin,
    {
        read_to_end(self, buf)
    }
}

pin_project! {
    #[derive(Debug)]
    #[must_use = "futures do nothing unless you `.await` or poll them"]
    pub struct ReadToEnd<'a, R: ?Sized> {
        reader: &'a mut R,
        buf: VecWithInitialized<&'a mut Vec<u8>>,
        // The number of bytes appended to buf. This can be less than buf.len() if
        // the buffer was not empty when the operation was started.
        read: usize,
        // Make this future `!Unpin` for compatibility with async trait methods.
        #[pin]
        _pin: PhantomPinned,
    }
}

impl<A> Future for ReadToEnd<'_, A>
where
    A: AsyncRead + ?Sized + Unpin,
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let me = self.project();

        read_to_end_internal(me.buf, Pin::new(*me.reader), me.read, cx)
    }
}
```

# Pin<Box<dyn Future<>>>解析

[Tower](https://github.com/tower-rs/tower)  是一个专注于对网络编程进行抽象的框架，最核心的抽象为 [Service](https://docs.rs/tower/latest/tower/trait.Service.html) trait。`Service::call` 接受一个 request 进行处理，成功则返回 response，否则返回 error。

```rust
fn call(Request) -> Result<Response, Error>
```

## Service trait 的定义

我们希望 Service 是异步编程风格，也就是 call 为 async 方法：

```rust
async fn call(Request) -> Result<Response, Error>
```

我们就可以在 `Service::cal`l 上 `await`：

```rust
service.call(request).await
```

然而当前 Rust 不支持 async trait 方法。

我们可以让 call 作为普通方法，返回一个实现了 Future 的类型：

```rust
trait Service<Request> {
    type Response;
    type Error;
    // ERROR: `impl Trait` not allowed outside of function and inherent
    // method return types
    fn call(&mut self, req: Request) -> impl Future<Output = Result<Self::Response, Self::Error>>;
}
```

还是不行，目前 Rust 也不支持在 Trait 中使用 `impl Trait` 做返回值，上一篇文章[<impl Trait 的使用>](https://mazhen.tech/impl_trait/)分析过原因。

Tower 在定义 [Service](https://docs.rs/tower/latest/tower/trait.Service.html) 时，使用了关联类型 `type Future` ，其实是将问题留给了 Service  的实现者，由用户选择 `type Future` 的实际类型：

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

## 实现自己的 Future 类型

我们在实现 Service 时，仍然需要为`type Future` 设置具体的类型。

既然没法让 call 直接返回 `impl Future`，一种方法是定义自己的 `Future` 类型，例如：

```rust
pub struct HttpRequest {
    url: String,
}
pub struct HttpResponse {
    code: u32,
}

pub struct ResponseFuture {
    request: HttpRequest,
}

impl Future for ResponseFuture {
    type Output = Result<HttpResponse, Error>;

    fn poll(self: Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> Poll<Self::Output> {
        println!("process url:{}", &self.as_ref().get_ref().request.url);
        Poll::Ready(Ok(HttpResponse { code: 200 }))
    }
}
```

在实现 Service 时，关联类型 `type Future` 设置为我们手工实现的 Future：

```rust
struct RequestHandler;

impl Service<HttpRequest> for RequestHandler {
    type Response = HttpResponse;

    type Error = Error;

    type Future = ResponseFuture;

    fn poll_ready(&mut self, cx: &mut std::task::Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: HttpRequest) -> Self::Future {
        ResponseFuture { request: req }
    }
}
```

然后就可以像正常的 Future 一样，使用 Service ：

```rust
#[tokio::main]
async fn main() {
    let mut service = RequestHandler {};
    match service
        .call(HttpRequest {
            url: "/user/mazhen".to_owned(),
        })
        .await
    {
        Ok(r) => println!("Response code: {}", r.code),
        Err(e) => println!("process failed. {:?}", e),
    }
}
```

## 使用 async blocks

手工实现 Future 会比较麻烦，我们一般都是用 `async fn`/`async blocks`语法糖生成 Future，那么这时 `Service::call` 返回什么类型呢？

```rust
struct RequestHandler;

impl Service<HttpRequest> for RequestHandler {
    ...
    type Future = ???

    fn call(&mut self, req: HttpRequest) -> Self::Future {
        async move {
            println!("process url {:?}", &req.url);
            Ok(HttpResponse { code: 200 })
        }
    }
}
```

既然不能返回 `impl Trait` ，可以让 call 返回 `trait object`，用 `trait object` 统一返回值的类型。

```rust
impl Service<HttpRequest> for RequestHandler {
    ...
    type Future = Box<dyn Future<Output = Result<Self::Response, Self::Error>>>;

    fn call(&mut self, req: HttpRequest) -> Self::Future {
        Box::new(async move {
            println!("process url {:?}", &req.url);
            Ok(HttpResponse { code: 200 })
        })
    }
}
```

这时候会报错，说 dyn Future 没有实现 `Unpin`：

```rust
    |
51  |     type Future = Box<dyn Future<Output = Result<Self::Response, Self::Error>>>;
    |                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Unpin` is not implemented for `(dyn std::future::Future<Output = Result<HttpResponse, std::io::Error>> + 'static)`
    |
```

编译错误提示的没有实现 `Unpin` trait 是什么意思？那要先从 Pin 说起。

## 一句话解释Pin

Pin 本质上解决的问题是保证 `Pin<P<T>>` 中的 T 不会被 move，除非 T 满足 T: Unpin。

## 什么是move

所有权转移的这个过程就是 move，例如：

```rust
let s1 = "Hello, world".to_owned();
let s2 = s1;
```

s2 浅copy s1 的内容，同时 String 的所有权转移给了 s2。

通过`std::mem::swap()`方法交换了两个可变借用 **&mut** 的内容，也发生了move。

## 为什么要Pin

自引用结构体，move了以后会出问题。

<img src="https://cdn.mazhen.tech/images/202209201529136.webp" alt="pin" style="zoom: 33%;" />

所以需要 Pin，不能move。

## 怎么 Pin 住的

保证 T 不会被move，需要避免两种情况：

* 不能暴露 T ，否则赋值、方法调用等都会move
* 不能暴露 &mut T，开发者可以调用 `std::mem::swap()` 或 `std::mem::replace()` 这类方法来 move 掉 T

`Pin<P<T>>`没有暴露T，而且没法让你获得 `&mut T`，所以就 Pin 住了T。但注意有个前提条件：T 没有实现 **Unpin**

## 谁没有实现 Unpin

`Unpin` 是一个auto trait，编译器默认会给所有类型实现 `Unpin`。唯独有几个例外，他们实现的是 `!Unpin`。

* PhantomPinned

```rust
/// A marker type which does not implement `Unpin`.
///
/// If a type contains a `PhantomPinned`, it will not implement `Unpin` by default.
#[stable(feature = "pin", since = "1.33.0")]
#[derive(Debug, Default, Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct PhantomPinned;

#[stable(feature = "pin", since = "1.33.0")]
impl !Unpin for PhantomPinned {}
```

* 如果你使用了 PhantomPinned，你的类型自动实现 `!Unpin`

```rust
use std::marker::PhantomPinned;

#[derive(Debug)]
struct SelfReference {
    name: String,
    // 在初始化后指向 name
    name_ptr: *const String,
    // PhantomPinned 占位符
    _marker: PhantomPinned,
}
```

* 编译器为 `async fn` 生成的匿名结构体实现的是 `!Unpin`

## async fn

`async fn` 是语法糖，在编译时，编译器使用[Generator](https://doc.rust-lang.org/std/ops/trait.Generator.html)为 `async fn` 生成匿名结构体，这个结构体实现了 `Future`。

这个匿名结构体是自引用的。原因是，如果 `async fn` 内有多个 `await`，执行到 `await` 可能因为资源没准备好而让出 CPU 暂停执行，随后该 `Future` 可能被调度到其他线程接着执行。所以这个匿名结构体需要保存跨 `await` 的数据，形成了自引用结构。

由于为 `async fn` 生成的结构体是自引用的，所以这个结构体实现了 `!Unpin`，表示它不能被 move。

这也是为什么不能给`Future::poll` 直接传 `&mut Self` 的原因：生成的匿名结构体不能被move，而拿到 `&mut Self`就可以使用 swap 或 replace之类的方法进行move，这样不安全，所以必须使用 `Pin<&mut Self>`。

## Future 都是 !Unpin 的吗

不一定。

`async fn` 语法糖生成的实现了 Future 的匿名结构，内部包含自引用，它会明确实现 `!Unpin`，不能 move。

但如果你自己实现的 Future，内部没有自引用，它就不是 `!Unpin`，当然可以 move。

也就是说，`Future` 和 `!Unpin` 是两个 trait，虽然它们经常联系在一起，但并不是实现了 `Future` 的类型都必须同时实现 `!Unpin`，没有包含自引用的 `Future`当然可以安全的 move 了。

## 实现了 Unpin 的 Future

如果是可以 move 的 Future，也就是实现了 Unpin 的 Future，在调用 `Future::poll` 的时候，要求传入 `Pin<&mut Self>`，会不会有什么问题呢？

首先，如果 `T：Unpin`，那么 `Pin<&mut T>` 就完全等同于 `&mut T`。换句话说，`Unpin` 意味着这个类型可以被移动，即使是在 Pin 住的情况下，所以 Pin 对这样的类型没有影响。因为 Pin 是智能指针，它实现了 `Deref/DerefMut`，只要满足 `T: Unpin`，你就能拿到`&mut T`：

```rust
#[stable(feature = "pin", since = "1.33.0")]
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```

其次，为了用户方便，[FutureExt](https://docs.rs/futures/latest/futures/future/trait.FutureExt.html) 提供了 `poll_unpin`，让你直接在 `Unpin` 的 `Future` 上 `poll`：

```rust
pub trait FutureExt: Future {
    /// A convenience for calling `Future::poll` on `Unpin` future types.
    fn poll_unpin(&mut self, cx: &mut Context<'_>) -> Poll<Self::Output>
    where
        Self: Unpin,
    {
        Pin::new(self).poll(cx)
    }
}
```

所以，如果你的 `Future` 是 `Unpin`，那么即使`Future::poll` 要求传入的是 `Pin<&mut Self>`，对你也没有任何影响。

## Box<dyn Futrue<>> 的问题

回到上面的问题，我们想让 `Service::call` 返回 `trait object`，也就是 `Box<dyn Futrue<>>`，会编译不过，为什么呢？

因为标准库为 Box 实现了 Future，但要求 Box 包装的 Future 必须同时实现了 Unpin ：

```rust
impl<F, A> Future for Box<F, A>
where
    F: Future + Unpin + ?Sized,
    A: Allocator + 'static, 
```

前面已经讲过，async 解语法糖生成的 Future 没有实现 Unpin，所以`Box::new(async{...})` 不满足类型约束，它没有实现 Future，不能在它上面await。

## Pin 实现了 Future

前面讲过，在 Future 上 poll 的时候，不能直接传入`&mut Self`，而需要传入 `Pin<&mut Self>`，需要这样调用 `Future::poll(Pin::new(&mut future), ctx)`。如果 Pin 实现了 Future ，我们就可以直接这样 poll 了： `Pin::new(&mut future).poll(ctx)`。

标准库也确实为 Pin 实现了 Future：

```rust
impl<P> Future for Pin<P>
where
    P: DerefMut,
    <P as Deref>::Target: Future, 
```

我们来看对 P 的约束，P 可解引用为 Future，也就是说，P是 Future 的引用 `&mut future`，或者是智能指针 `Box<dyn Future>` 都可以满足约束。因为 `Pin<Box<dyn Future<>>>` 实现了 Future，我们可以用它作为 `Service::call` 返回值类型。

Box 提供了 pin 方法，让用户构建 `Pin<Box<T>>`：

```rust
pub fn pin(x: T) -> Pin<Box<T, Global>>
```

使用 `Box::pin`，RequestHanlder 可以这么修改：

```rust
impl Service<HttpRequest> for RequestHandler {
    ...
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>>>>;

    fn call(&mut self, req: HttpRequest) -> Self::Future {
        Box::pin(async move {
            println!("process url {:?}", &req.url);
            Ok(HttpResponse { code: 200 })
        })
    }
}
```

这回终于可以了。事实上，在异步场景下，我们经常会看到使用 `Box::pin` 去包装 `async block`。

## Pin<Box<dyn Future<>>>

`Pin<Box<dyn Future<>>>` 除了实现了Future，也实现了 Unpin。

因为 Pin 实现了 Unpin，只要 P 是 Unpin 的：

```rust
impl<P> Unpin for Pin<P>
where
    P: Unpin
```

而 Box 正好是 Unpin 的：

```rust
impl<T, A> Unpin for Box<T, A>
where
    A: Allocator + 'static,
    T: ?Sized,
```

因此 `Pin<Box<T>>`是 Unpin 的。可以这么理解，Pin 钉住了 T，但 Pin 本身是 Unpin的，可以安全的 move。

很多异步方法需要你的 `Future` 同时实现了 `Unpin` ，例如[tokio::select!()](https://docs.rs/tokio/latest/tokio/macro.select.html)，而 `async fn` 返回的 `Future` 显然不满足 `Unpin`，这个时候仍然可以用 `Box::pin`把 Future pin 住，得到的`Pin<Box<dyn Future<...>>>` 同时实现了 Future 和 Unpin，满足你的要求。

简单总结，在异步编程场景，我们经常会用`Box::pin` 包装 `async block`，获得同时实现了 Future 和 Unpin 的`Pin<Box<dyn Future<>>>`。

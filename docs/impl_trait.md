# impl Trait 的使用

Rust 通过 [RFC conservative impl trait](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) 增加了新的语法 `impl Trait`，它被用在函数返回值的位置上，表示返回的类型将实现这个 Trait。随后的 [RFC expanding impl Trait](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md) 更进一步，允许 `impl Trait` 用在函数参数的位置，表示由调用者决定参数的具体类型，其实就等价于函数的泛型参数。

## impl Trait 作为函数参数

根据  [RFC on expanding impl Trait](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md)， `impl Trait` 可以用在函数参数中，作用是作为函数的匿名泛型参数。

> Expand `impl Trait` to allow use in arguments, where it behaves like an anonymous generic parameter.

也就是说，`impl Trait` 作为函数参数，和泛型参数是等价的：

```rust
// These two are equivalent
fn map<U>(self, f: impl FnOnce(T) -> U) -> Option<U>
fn map<U, F>(self, f: F) -> Option<U> where F: FnOnce(T) -> U
```

不过，`impl Trait`和泛型参数有一个不同的地方，`impl Trait` 作为参数，不能明确指定它的类型：

```rust
fn foo<T: Trait>(t: T)
fn bar(t: impl Trait)

foo::<u32>(0) // this is allowed
bar::<u32>(0) // this is not
```

除了这个差别，可以认为`impl Trait` 作为函数参数，和使用泛型参数是等价的。

## impl Trait 作为函数返回值

 `impl Trait` 作为函数的返回值，表示返回的类型将实现这个 Trait。

```rust
fn foo(n: u32) -> impl Iterator<Item = u32> {
    (0..n).map(|x| x * 100)
}
fn main() {
    for x in foo(10) {
        println!("{}", x);
    }
}
```

在这种情况下，需要注意函数的所有返回路径必须返回完全相同的具体类型。

```rust
// 编译错误，即使这两个类型都实现了Bar
fn f(a: bool) -> impl Bar {
    if a {
        Foo { ... }
    } else {
        Baz { ... }
    }
}
```

可以把函数返回值位置的 `impl Trait` 替换为泛型吗？

```rust
// 不能编译
fn bar<T: Iterator<Item = u32>>(n: u32) -> T {
    (0..n).map(|x| x * 100)
}
```

编译器给的错误信息是，期待返回值的类型是泛型类型 T，却实际却返回了一个具体类型。编译器很智能的给出了使用 `impl Iterator<Item = u32>`作为返回类型的建议：

```rust
 --> src/main.rs:6:5
  |
5 | fn bar<T: Iterator<Item = u32>>(n: u32) -> T {
  |        -                                   -
  |        |                                   |
  |        |                                   expected `T` because of return type
  |        this type parameter                 help: consider using an impl return type: `impl Iterator<Item = u32>`
6 |     (0..n).map(|x| x * 100)
  |     ^^^^^^^^^^^^^^^^^^^^^^^ expected type parameter `T`, found struct `Map`
  |
  = note: expected type parameter `T`
                     found struct `Map<std::ops::Range<u32>, [closure@src/main.rs:6:16: 6:27]>`
```

## Universals vs. Existentials

在  [RFC on expanding impl Trait](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md) 中使用了两个术语，Universal 和 Existential：

> - Universal quantification, i.e. "for any type T", i.e. "caller chooses". This is how generics work today. When you write `fn foo<T>(t: T)`, you're saying that the function will work for any choice of `T`, and leaving it to your caller to choose the `T`.
> - Existential quantification, i.e. "for some type T", i.e. "callee chooses". This is how `impl Trait` works today (which is in return position only). When you write `fn foo() -> impl Iterator`, you're saying that the function will produce some type `T` that implements `Iterator`, but the caller is not allowed to assume anything else about that type.

简单来说：

- `impl Trait` 用在参数位置是 universal type，也就是泛型类型，它可以是任意类型，由函数的调用者指定具体的类型。

- `impl Trait` 用在返回值位置是 existential type，它不能是任意类型，而是由函数的实现者指定，一个实现了 Trait 的具体类型。调用者不能对这个类型做任何假设。

也就是说，`impl Trait` 用在返回位置不是泛型，编译时不需要单态化，抽象类型可以简单地替换为调用代码中的具体类型。

## 在 Trait 中使用 impl Trait

Rust 目前还不支持在 Trait 里使用 `impl Trait` 做返回值：

```rust
trait Foo {
    // ERROR: `impl Trait` not allowed outside of function and inherent
    // method return types
    fn foo(&self) -> impl Iterator<Item=u8>;
}
```

因为 `impl Trait` 用在返回值位置是 existential type，意味着这个函数将返回一个实现了这个 Trait 的单一类型，而函数定义在 Trait 中，意味着每个实现了 Trait 的类型，都可以让这个函数返回不同类型，对编译器来说这很难处理，因为它需要知道被返回类型的具体大小。

一个简单的解决方法是让函数返回 `trait object`：

```rust
trait Foo {
    fn foo(&self) -> Box<dyn Iterator<Item=u8>>;
}
```

带有 `trait object` 的函数不是泛型函数，它只带有单一类型，这个类型就是 `trait object` 类型。`Trait object` 本身被实现为胖指针，其中，一个指针指向数据本身，另一个则指向虚函数表（vtable）。

这样定义在 Trait 中的函数，返回的不再是泛型，而是一个单一的  `trait object` 类型，大小固定（两个指针大小），编译器可以处理。

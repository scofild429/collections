---
title: "Rust"
org_id: "98595F3E-F152-41EB-96FD-99AB77270033"
---

# Rust

Examples use Rust edition 2021. Each numbered module example is a separate project; file comments identify where each code block belongs.

<a id="14572E13-097B-413F-8982-33BA0F804B65"></a>

## Modules, visibility, and imports

| Syntax | Meaning |
|---|---|
| `mod garden { ... }` | Define an inline module |
| `mod garden;` | Declare a module whose body is in another file |
| `pub mod garden;` | Declare that module with public visibility |
| `use path::Item;` | Bring an existing item into the current scope |
| `pub use path::Item;` | Re-export an accessible item under a public path |

Items are private by default. A private item is accessible in its defining module and its descendants. Making a module public does not make every item inside it public. A public re-export can expose a public item through a shorter path while its implementation module stays private. See [Rust visibility and privacy](https://doc.rust-lang.org/reference/visibility-and-privacy.html).

### 1. Inline module

```rust
// src/main.rs
mod garden {
    pub fn test() {}
}

fn main() {
    garden::test();
}
```

The root can access its child `garden`, but `test` must be visible from outside `garden`. Use `pub`, not `pb`; function and inline-module definitions do not end with a semicolon. See [module syntax](https://doc.rust-lang.org/reference/items/modules.html).

### 2. Module in a separate file

```text
src/
├── main.rs
└── network.rs
```

```rust
// src/network.rs
pub fn test() {}
```

```rust
// src/main.rs
mod network;

fn main() {
    network::test();
}
```

`network.rs` contains the module body, so do not wrap it in another `mod network { ... }`. It can still declare its own child modules. A file is not automatically part of the module tree: the parent declaration connects it. The usual alternatives are `network.rs` or `network/mod.rs`, but not both for the same module. See [separating modules into files](https://doc.rust-lang.org/book/ch07-05-separating-modules-into-different-files.html).

### 3. Child modules in a matching directory

```text
src/
├── main.rs
├── dep.rs
└── dep/
    ├── submod1.rs
    └── submod2.rs
```

```rust
// src/dep/submod1.rs
pub fn test() {}
```

```rust
// src/dep/submod2.rs
// Reserved for another child module.
```

```rust
// src/dep.rs
pub mod submod1;
pub mod submod2;
```

```rust
// src/main.rs
mod dep;

fn main() {
    dep::submod1::test();
}
```

The extension is `.rs`, not `.rf`. The directory holds child modules; `dep.rs` declares them.

### 4. `use` shortens a path

Keep the files from example 3 and replace `src/main.rs` with:

```rust
// src/main.rs
mod dep;
use crate::dep::submod1::test;

fn main() {
    test();
}
```

`use` does not declare a module or bypass privacy. `crate::` starts at the current crate root; `self::` starts at the current module; `super::` refers to its parent. See [bringing paths into scope](https://doc.rust-lang.org/book/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html).

### 5. Re-export from a private implementation module

```text
src/
├── main.rs
├── local.rs
└── local/
    ├── dep.rs
    └── dep/
        ├── submod1.rs
        └── submod2.rs
```

```rust
// src/local/dep/submod1.rs
pub fn test() {}
```

```rust
// src/local/dep/submod2.rs
// Reserved for another child module.
```

```rust
// src/local/dep.rs
mod submod1;
mod submod2;
pub use self::submod1::test;
```

```rust
// src/local.rs
pub mod dep;
```

```rust
// src/main.rs
mod local;
use crate::local::dep::test;

fn main() {
    test();
}
```

The caller uses `local::dep::test`. It cannot name `local::dep::submod1::test` from the root because `submod1` is private. The public re-export exposes the function without exposing that module path.

### 6. A package with a library and a binary

Use the library name `my_app` consistently. In a normal Cargo package, `src/lib.rs` and `src/main.rs` are roots of **separate crates**.

```toml
# Cargo.toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2021"
```

```text
src/
├── main.rs
├── lib.rs
├── dep.rs
└── dep/
    ├── submod1.rs
    └── submod2.rs
```

```rust
// src/dep/submod1.rs
pub fn test() {}
```

```rust
// src/dep/submod2.rs
// Reserved for another child module.
```

```rust
// src/dep.rs
mod submod1;
mod submod2;
pub use self::submod1::test;
```

```rust
// src/lib.rs
pub mod dep;
```

```rust
// src/main.rs
use my_app::dep::test;

fn main() {
    test();
}
```

The binary imports through the library crate name, not `mod lib;`. By default that name follows the package name with hyphens replaced by underscores; `[lib] name` can override it.

<a id="E7B0165D-C715-4A8C-B10E-717BA1C91FED"></a>

## Ownership and types / 所有权与类型

### `Copy`：复制与移动

赋值是否复制取决于类型是否实现 `Copy`，不能简单概括为“基本类型都复制”。整数、浮点数、`bool`、`char` 和共享引用 `&T` 实现 `Copy`；`String`、`Vec<T>` 和可变引用 `&mut T` 不实现。数组 `[T; N]` 在 `T: Copy` 时也实现 `Copy`。

```rust
fn main() {
    let a = 42;
    let b = a; // i32: Copy
    assert_eq!(a, b); // a remains usable

    let first = String::from("hello");
    let second = first; // Move ownership
    // println!("{first}"); // Would fail: first has moved
    assert_eq!(second, "hello");
}
```

`Copy` 是隐式复制；`Clone` 是显式的 `.clone()` 操作。自定义类型可以在满足条件时实现或派生 `Copy`，但必须同时实现 `Clone`，所有字段必须是 `Copy`，且该类型不能实现 `Drop`。参见 [Copy 文档](https://doc.rust-lang.org/std/marker/trait.Copy.html)。

### 切片 `[T]` 与切片引用 `&[T]`

| 写法 | 含义 |
|---|---|
| `[T; N]` | 数组；长度 `N` 是类型的一部分 |
| `[T]` | 切片；大小在编译时不固定，是动态大小类型（DST） |
| `&[T]` | 共享切片引用；包含数据指针和长度信息 |
| `&mut [T]` | 可变切片引用，可修改元素 |

`[]` 本身不是“超类型”。`[T]` 的 `T` 表示元素类型；长度不是切片类型的一部分，但仍保留在切片引用的运行时元数据中。切片不拥有底层元素。

```rust
fn total(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn main() {
    let values = [10, 20, 30, 40];
    let slice: &[i32] = &values[1..3];
    assert_eq!(slice.len(), 2);
    assert_eq!(total(slice), 50);
    assert_eq!(total(&values), 100);
}
```

参见 [Rust 切片类型](https://doc.rust-lang.org/reference/types/slice.html)。

### `Self`、`self`、`&self`、`&mut self`

| 写法 | 含义 |
|---|---|
| `Self` | `impl` 中表示被实现的类型；trait 中表示实现该 trait 的类型 |
| `self` | 按值接收实例；非 `Copy` 类型通常会被移动进方法 |
| `&self` | 共享借用实例 |
| `&mut self` | 可变借用实例 |

`Self` 是类型，不是自动生成实例的操作。可以用 `Self { ... }` 构造结构体，也可以在返回类型中写 `Self`。`self` 并不一定“转换”实例；它只是接收者，具体行为由方法决定。对于 `Copy` 类型，按值调用可以复制接收者。

```rust
struct Counter {
    value: i32,
}

impl Counter {
    fn new() -> Self {
        Self { value: 0 }
    }

    fn value(&self) -> i32 {
        self.value
    }

    fn increment(&mut self) {
        self.value += 1;
    }

    fn into_value(self) -> i32 {
        self.value
    }
}

fn main() {
    let mut counter = Counter::new();
    counter.increment();
    assert_eq!(counter.value(), 1);
    assert_eq!(counter.into_value(), 1); // Consumes counter
}
```

参见 [Rust 方法语法](https://doc.rust-lang.org/book/ch05-03-method-syntax.html)。

### Trait 不会普遍从字段“继承”

字段都实现某个 trait，不代表外层结构体或枚举自动实现该 trait。

- 对支持派生的 trait，例如 `Debug`、`Clone`、`PartialEq`，可以使用 `#[derive(...)]`，前提是生成的实现满足所需约束。
- 普通自定义 trait 通常需要显式 `impl`；不能对任意 trait 直接写 `derive`，除非有对应的派生宏。
- `Send`、`Sync` 等 **auto traits** 有专门的自动实现规则，不能推广到所有 trait。

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let first = Point { x: 1, y: 2 };
    let second = first;
    assert_eq!(first, second);
}
```

参见 [Copy 的派生条件](https://doc.rust-lang.org/std/marker/trait.Copy.html)与 [auto traits](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)。

## Trait objects / 特征对象

### 泛型与动态分发

`Vec<T>` 在一次具体使用中只有一种元素类型；`T: Draw` 可以约束该类型。要在一个集合中保存不同实现类型，可以使用 `Vec<Box<dyn Draw>>`。

`dyn Draw` 是动态大小类型，通常通过 `&dyn Draw`、`&mut dyn Draw`、`Box<dyn Draw>` 等指针使用。`Box` 是拥有所有权的智能指针，不是借用引用。

特征对象指针包含数据指针和 vtable 信息，调用通过动态分发完成。vtable 涉及该 trait 及其 supertraits 的可分发方法；不能简单说“只包含该 trait 的方法”，也不应依赖其具体内存布局。参见 [trait object 类型](https://doc.rust-lang.org/reference/types/trait-object.html)。

```rust
trait Draw {
    fn draw(&self) -> &'static str;
}

struct Button;
struct Label;

impl Draw for Button {
    fn draw(&self) -> &'static str {
        "button"
    }
}

impl Draw for Label {
    fn draw(&self) -> &'static str {
        "label"
    }
}

fn main() {
    let widgets: Vec<Box<dyn Draw>> = vec![Box::new(Button), Box::new(Label)];
    let names: Vec<_> = widgets.iter().map(|widget| widget.draw()).collect();
    assert_eq!(names, ["button", "label"]);
}
```

### Dyn compatibility（旧称 object safety）

“不能返回 `Self`、不能有泛型方法”只是部分规则，不能作为完整判断标准。主要要求包括：

- trait 不能要求 `Self: Sized`，且所有 supertraits 必须兼容 `dyn`。
- 不能包含关联常量，也不能有带泛型参数的关联类型。
- 可动态调用的方法不能有类型泛型参数，但允许生命周期参数。
- 可动态调用的方法必须使用受支持的接收者，例如 `&self`、`&mut self` 或 `self: Box<Self>`；不能在其他参数或返回类型中使用 `Self`。
- 可动态调用的方法不能返回 opaque type，例如 `async fn` 隐式返回的 future 或返回位置的 `impl Trait`。

不满足方法分发要求的方法可以用 `where Self: Sized` 排除在动态调用接口之外，而不必使整个 trait 失去 dyn compatibility。普通关联类型也不等于“不兼容”，例如 `dyn Iterator<Item = i32>` 可以指定关联类型。

```rust
trait Draw {
    fn draw(&self);

    fn new() -> Self
    where
        Self: Sized;
}

struct Button;

impl Draw for Button {
    fn draw(&self) {}

    fn new() -> Self {
        Self
    }
}

fn main() {
    let button = Button::new();
    let drawable: &dyn Draw = &button;
    drawable.draw();
    // new() is available on Button, not through the trait object.
}
```

完整规则见 [Rust Reference：dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)。

## Source provenance

<!-- Org properties: {"craft_id": "CD93D283-4A82-4622-99E1-8646874AAE32", "imported": "2026-09-27"} -->

Original import: [Rust in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=CD93D283-4A82-4622-99E1-8646874AAE32), imported 2026-09-27. The duplicated imported notes have been consolidated into the sections above. The Craft link is retained as provenance; this review used the local note and linked official documentation.

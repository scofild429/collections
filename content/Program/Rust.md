---
title: "Rust"
org_id: "98595F3E-F152-41EB-96FD-99AB77270033"
---

# Rust

<a id="14572E13-097B-413F-8982-33BA0F804B65"></a>

## rust pub use mod

### 1. in the same file

mod code and the calling in the same file, can direct all it.

``` rust
// main.rs
mod garden{
  pb fn test(){};
};
fn main () {
    garden::test();
}

```

### 2. file instead of mod

put all rust code in a file, no mod key word needed in this file. We can put this file as a mod in other file (in the same directory) by using mod file name without .rs postfix.

``` rust
// network.rs no mod key word
pub fn test(){};
// main.rs
mod network;
fn main() {
    network::test();
}
```

### 3. file and folder as mod

file is still used as case 2 as modular, include sub modular files in the same named folder

``` rust
// dep/submod1.rs no mod key word
pub fn test(){};
// dep/submod2.rf no mod key word
// dep.rs 
pub mod submod1;
pub mod submod2;
//main.rs
mod dep;
fn main () {
  dep::submod1::test();
}
```

### 4. use for shortcuts

in the 3. case we shortcut the =dep::submod1::test() as

``` rust
// ....
// main.rs 
mod dep;
use dep::submod1::test;
fn main () {
    test()
}

```

### 5. Re-exporting with pub use in a closed system,

#### nested folder

    src/
    ├── main.rs
    ├── local.rs
    └── local/
        ├── dep.rs
        └── dep/
            ├── submod1.rs
            └── submod2.rs

``` rust
// --- src/local/dep/submod1.rs ---
pub fn test() {}

// --- src/local/dep/submod2.rs ---
// (Empty or other code)

// --- src/local/dep.rs ---
mod submod1;
mod submod2;

// Perfect! You re-exported `test`
pub use self::submod1::test; 

// --- src/local.rs ---
// You must declare `dep` here so `local` knows it exists.
pub mod dep; 

// --- src/main.rs ---
// 1. Declare the `local` module to attach `local.rs` to the tree
mod local; 

// 2. Bring your re-exported function into scope
use local::dep::test;

fn main() {
    test(); // It works!
}

```

**\***

#### in a library called `my_app`

    src/
    ├── main.rs
    ├── lib.rs            <-- This acts as the root of your library
    ├── dep.rs
    └── dep/
        ├── submod1.rs
        └── submod2.rs

``` rust
// --- src/dep/submod1.rs ---
pub fn test() {}

// --- src/dep.rs ---
mod submod1;
mod submod2;
pub use self::submod1::test;

// --- src/lib.rs ---
// This file IS the root of your library. 
// You don't need `pub mod local;`, you just expose your internal modules directly.
pub mod dep;

// --- src/main.rs ---
// Because main.rs and lib.rs are separate, main.rs must import the library
// using the crate name from Cargo.toml ("my_awesome_app").
use my_awesome_app::dep::test;

fn main() {
    test(); 
}
```

\* \*

<a id="E7B0165D-C715-4A8C-B10E-717BA1C91FED"></a>

## Notes

- **基本类型在 Rust 中赋值是以 Copy 的形式**

- \*切片和切片引用\*： \[\]是超类型，可以有任意的长度类型，&\[\]是使用任意长度类型\[\]的操作手柄，而&\[T\]范型则拓展这个规则到任意的代指类型（长度擦除（Unsized） + 指针间接访问（Reference） + 类型参数化（Generics）

- Self and self

  - Self在impl和srait中代指相应类型并可生成其实例。
  - self在impl和trait中转化实例，原实例被消费
  - &self和& mut self用于常见的实例访问和修改

- in RUST, if all the fields of a structure(enum) has some kind of Trait, so the structure(enum) can also has this Trait(need deriver if is not elemental type of Rust).

- 如果只需要同质（相同类型）集合，可以采用泛型+特征约束这种写法,

  - **特征对象几乎总是使用引用的方式** ，如 =&dyn Draw=、=Box\<dyn Draw\>=

  - **=btn= 是哪个特征对象的实例，它的 =vtable= 中就只包含了该特征的方法**

  - 不是所有特征都能拥有特征对象，只有对象安全的特征才行。当一个特征的所有方法都有如下属性时，它的对象才是安全的：

    - 方法的返回类型不能是 `Self`
    - 方法没有任何泛型参数

## Craft import — Rust — 2026-09-27

<!-- Org properties: {"craft_id": "CD93D283-4A82-4622-99E1-8646874AAE32", "imported": "2026-09-27"} -->

Source: [Rust in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=CD93D283-4A82-4622-99E1-8646874AAE32)

### 切片和切片引用：

- \[\]是超类型，可以有任意的长度类型，
- &\[\]是使用任意长度类型\[\]的操作手柄，
- 而&\[T\]范型则拓展这个规则到任意的代指类型
  - 长度擦除（Unsized）
  - 指针间接访问（Reference）
  - 类型参数化（Generics）

### COPY

- 基本类型在 Rust 中赋值是以 Copy 的形式

------------------------------------------------------------------------

### SELF

- Self在impl和trait中代指相应类型并可生成其实例。
- self在impl和trait中转化实例，原实例被消费
- &self和& mut self用于常见的实例访问和修改

------------------------------------------------------------------------

### In RUST, if all the fields of a structure or enum has some kind of Trait, so this structure or enum can also has (inherit) this Trait.

This need deriver if it is not elemental type of Rust.

### **特征对象**

如果只需要同质（相同类型）集合, 可以采用泛型+特征约束这种写法, 特征对象几乎总是使用引用的方式，如

``` text
&dyn Draw
Box<dyn Draw>
```

对于其实例的使用，是哪个特征对象的实例，它的 vtable 中就只包含了该特征的方法

------------------------------------------------------------------------

不是所有特征都能拥有特征对象，只有对象安全的特征才行。当一个特征的所有方法都有如下属性时，它的对象才是安全的：

- 方法的返回类型不能是 Self
- 方法没有任何泛型参数

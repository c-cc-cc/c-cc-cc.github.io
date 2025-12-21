---
title: 深入理解 Rust 闭包
date: 2025-12-21T11:00:00.000Z
updated: 2025-12-21T09:50:44.749Z
tags: [rust]
categories: [rust]
---



> 闭包是一个很常见的语言特性，很多语言都有，也都很容易理解，Rust 闭包因为 所有权&生命周期 的存在，导致存在 `FnOnce`、`FnMut`、`Fn` 三种 `Trait`，增加了一些理解上的复杂性，尤其是存在一些反直觉的情况，本文想探讨此问题。

Rust 闭包实际是一种 `匿名结构体`，一旦声明，编译器就会创建一个新的 `结构体`，这个 `结构体` 会包含所有捕获的 `自由变量`，这个 `结构体` 不可见因此无法被其它地方使用。

Rust 闭包因为受到 `所有权` 和 `借用` 规则的限制，闭包在捕获 `自由变量` 时，会根据闭包体的操作为 `闭包结构体` 实现不同的 `Trait`：`FnOnce` `FnMut` `Fn`，理解为什么会需要这三种 `Trait` 及其关联关系 是理解并掌握 Rust 闭包的关键一环。如果没有理解其本质，实际使用时会是比较迷惑的。三个 `Trait` 的区别如下：

1. `FnOnce`：闭包调用时会消耗 `self`（获取所有权），只能调用一次。典型场景：闭包将捕获的变量移动（move）到其他地方。
2. `FnMut`：闭包调用时需要 `&mut self`，可以修改捕获的变量，可以被调用多次。
3. `Fn`：闭包调用时只需要 `&self`，不修改捕获的变量，可以被调用多次。

需要注意的是，这三个 `Trait` 描述的是**闭包被调用时的行为**，而不是**闭包捕获变量的方式**。闭包如何捕获变量（按值、按不可变引用、按可变引用）由编译器根据闭包体的使用情况自动推断，而实现哪个 `Trait` 则取决于闭包体对捕获变量的操作。

接下来，尝试从 `Trait` 实现及底层权限模型角度，分析其本质。

以下为三个 `Trait` 的定义及一段简单的示例代码：

```rs
pub trait FnOnce<Args>
where
    Args: Tuple,
{
    type Output;
    // Required method
    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args>: FnOnce<Args>
where
    Args: Tuple,
{
    // Required method
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args>: FnMut<Args>
where
    Args: Tuple,
{
    // Required method
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}
```

Ref：

* <https://doc.rust-lang.org/std/ops/trait.FnOnce.html>
* <https://doc.rust-lang.org/std/ops/trait.FnMut.html>
* <https://doc.rust-lang.org/std/ops/trait.Fn.html>

再看一段示例代码：

```rs
fn call_fnonce<F: FnOnce()>(f: F) {
    f();
}

fn call_fnmut<F: FnMut()>(mut f: F) {
    f();
}

fn call_fn<F: Fn()>(f: F) {
    f();
}

fn main() {
    let x = 10;
    let sss = String::from("hello world");

    let fn_closure = || println!("Fn: {}, {}", x, sss); // Fn 闭包

    // 自动转换：Fn → FnMut → FnOnce
    call_fn(fn_closure);        // ✅ Fn 作为 Fn
    call_fnmut(fn_closure);     // ✅ Fn 作为 FnMut
    call_fnonce(fn_closure);    // ✅ Fn 作为 FnOnce

    fn_closure(); //  ✅ Fn 可以再次被调用

    // FnMut 闭包
    let mut y = 5;
    let mut fnmut_closure = || {
        y += 1;
        println!("FnMut: {}", y);
    };

    call_fnmut(&mut fnmut_closure);  // ✅ FnMut 作为 FnMut
    call_fnonce(&mut fnmut_closure); // ✅ FnMut 作为 FnOnce
    
    fnmut_closure(); // ✅ 因为使用了引用传递，闭包本身未被移动，仍可调用
    
    // call_fn(&mut fnmut_closure);  // ❌ FnMut 不能作为 Fn
}
```

从以上 `Trait` 定义上来看，`Fn` 依赖 `FnMut`，而 `FnMut` 又依赖 `FnOnce`，如果一个闭包被编译器归类为 `Fn`，那么编译器也会同时为它实现 `FnMut` 和 `FnOnce`。

这个依赖关系意味着 Rust 编译器 会为 `fn_closure` 闭包同时实现 `Fn`、`FnMut`、`FnOnce` 三个 `Trait`，也就是 `fn_closure` 对应的结构体实现了 `call` `call_mut` 和 `call_once` 三个关联函数。

这里会令人感到困惑，因为 `fn_closure` 的实现代码以只读的方式捕获了变量 `x` 并没有以可变引用或转移所有权方式捕获变量，为什么要为其实现 `FnMut` 和 `FnOnce`，只实现 `Fn` 不就够了吗？

这里困惑的来源两点：

1. `fn_closure` 闭包实现代码 与 编译器为其生成的 `Trait` 是不对应的，额外实现了另外两个 `Trait`。
2. `fn_closure` 闭包可以传递给要求 `FnMut` 和 `FnOnce` 参数的函数，为什么一个捕获只读引用的闭包 `Fn` 要允许被当做捕获可变引用的闭包 `FnMut` 使用？或者需要转移所有权的闭包 `FnOnce` 使用？只读 为什么可以传给 可变的？可变的 为什么可以传给 可移动？

要想理解这个问题，可以从编译器如何实现 `Fn`、`FnMut`、`FnOnce` 这三个 Trait 的调用方法入手。

对于以下包含闭包的代码：

```rs
let x = 10;
let fn_closure = || println!("Fn: {}", x);
```

编译器会为 `fn_closure` 生出成类似如下的闭包结构体和 `Trait` 实现：

```rs
// 编译器生成的闭包结构
struct Closure {
    x: i32,  // 捕获 x 的副本（i32 实现了 Copy）
}

impl Fn<()> for Closure {
    fn call(&self, _args: ()) {
        println!("Fn: {}", self.x);  // 只读访问
    }
}

impl FnMut<()> for Closure {
    fn call_mut(&mut self, args: ()) {
        self.call(args) // 只是转发到 Fn::call
    }
}

impl FnOnce<()> for Closure {
    type Output = ();
    
    fn call_once(self, args: ()) -> () {
        self.call(args) // 只是转发到 Fn::call
    }
}
```

Ref: https://doc.rust-lang.org/src/core/ops/function.rs.html#253

从以上实现中，可以看到 `call_mut` 和 `call_once` 方法只是简单地转发调用到 `call` 方法上，最终 `call` 方法只读访问了捕获的变量 `x`。从 Rust 的所有权和借用规则来看，这样的设计实现是满足相关约束的。

* `call(&self)`：通过不可变引用调用，完全安全
* `call_mut(&mut self)`：即使有可变引用，闭包也不会修改任何东西，所以安全
* `call_once(self)`：即使获取所有权，闭包也不会被"消耗"，可以安全调用

从调用方法的转发实现可以看到：`call_mut` 和 `call_once` 都转发到 `call`，转发方向是 **`FnMut` → `Fn`** 和 **`FnOnce` → `Fn`**。

这里与通常理解的方向是相反的，根据定义，`Fn` 包含的能力是最全的，这里很容易用 "类型转换" 的方式理解：`Fn` 可以 "类型转换" 成 `FnMut` 或 `Fn`。
实际上，这种理解是**错误**的，这里并不存在 "类型转换"，并且这样 "类型转换" 方式理解的方向是相反的，这也是为什么这里容易产生困惑的原因。

## 本质：约束与包含

为了彻底消除这种反直觉的困惑，我们需要调整观察的视角：不要把 `Fn`、`FnMut`、`FnOnce` 理解为闭包拥有的 "能力强度"，而应该理解为对调用者施加的 **"约束强度"**。

* **Fn**：约束最**严格**。调用者必须保证 "不仅能调用，而且调用后闭包必须保持原样，状态不能变，所有权不能丢"。
* **FnOnce**：约束最**宽松**。调用者唯一需要保证的是 "可以调用它"，至于调用后闭包还在不在，调用者不关心。

这是一种**包含关系**，而不是等级关系：

* 所有满足 `Fn` 约束（不可变借用调用）的闭包，**天然满足** `FnMut` 约束（因为不可变借用是一种特殊的可变借用，即 "不修改的可变借用"）。
* 所有满足 `FnMut` 约束（可变借用调用）的闭包，**天然满足** `FnOnce` 约束（因为借用是所有权转移的一种特殊情况，即 "借用后归还"）。

因此：**`Fn` ⊂ `FnMut` ⊂ `FnOnce`**

这就解释了为什么编译器会自动为 `Fn` 实现 `FnMut` 和 `FnOnce`：这不是 "类型转换"，而是 **"子集归属"**。

就像现实生活中的例子：

* **Fn (能跑马拉松)**：要求身体素质极高。
* **FnMut (能散步)**：要求身体素质一般。
* **FnOnce (能活着)**：要求最低。

一个能跑马拉松的人（`Fn`），虽然他拥有最高的能力，但他完全可以去散步（当做 `FnMut` 用），也完全可以只是活着（当做 `FnOnce` 用）。你不需要对他进行“改造”或“转换”，因为他**本来就是**一个能散步、能活着的人。


---

<a href="https://github.com/liuyanjie/knowledge/tree/master/langs/rust/rust-closure.md" >查看源文件</a>&nbsp;&nbsp;<a href="https://github.com/liuyanjie/knowledge/edit/master/langs/rust/rust-closure.md">编辑源文件</a>

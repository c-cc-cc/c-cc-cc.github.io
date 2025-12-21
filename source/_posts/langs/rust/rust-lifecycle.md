---
title: 深入理解 Rust 生命周期
date: 2025-12-21T11:00:00.000Z
updated: 2025-12-21T09:50:44.748Z
tags: [rust]
categories: [rust]
---



> Rust 生命周期是 Rust 最与众不同的语言特性之一，也是 Rust 最难理解的语言特性之一。本文尝试解释 Rust 为什么需要生命周期标注。

## 函数参数进行生命周期标注

Q: 为什么需要对函数参数进行生命周期标注？

虽然 Rust 编译器可以根据变量的声明位置和代码上下文逻辑推断出变量及其引用的生命周期及依赖关系。但是如果一个引用类型的变量来自于某个函数的返回值时，编译器将难以推导出该引用在什么作用域范围内使用是安全的以及实际使用的范围是否是安全的，也就无判断是否满足 Rust 借用检查器的作用域规则。为了解决这一问题，Rust 给出了生命周期标注的解决方案。

所以，生命周期标注主要解决编译器无法追踪的函数返回引用类型变量时，生命周期信息中断的问题。最终，确保引用在使用时是有效的。
生命周期标注的目的是帮助编译器理解引用的有效性范围，从而避免悬垂引用和数据竞争。

> **补充说明：生命周期省略规则 (Lifetime Elision Rules)**
> 虽然 Rust 要求显式标注复杂的生命周期，但对于常见模式（如单参数单返回值），编译器内置了“生命周期省略规则”来自动推导。只有当编译器无法利用这些规则确定引用关系时，才需要手动标注。

Q: 为什么 Rust 编译器无法推导或追踪跨函数的情况？

根本原因在于：**Rust 编译器是独立对每个函数进行编译的**。

1. 编译器在编译被调函数时，并不知道该函数将会被如何以及何时被调用，当然也不知道传入参数的实际生命周期是怎样的。参数的实际生命周期取决于上下文，并不是固定不变的。
2. 编译器在编译主调函数时，同样也可能不知道被调函数是如何实现的，它只是一个黑盒。编译器在编译时只需要知道函数的签名（即函数的名称、参数类型和返回类型），而不需要知道函数的具体实现细节。实际在链接阶段，链接器才会将主调函数和被调函数的二进制链接在一起。

除了以上两个核心因素，编译器进行默认的推导可能带来 歧义、灵活性降低、隐式难以琢磨理解 等其他问题。

Q: 生命周期标注是如何解决此问题的？

生命周期标注建立了函数的出参和入参的生命周期之间的关系，即建立了函数返回值赋值的本地变量和作为函数参数主调函数的其他本地变量之间的关系，这使得编译器能够根据这个关系推断出函数返回后的本地变量的作用域范围是怎样的，Rust 借用检查器的作用域规则也就可以进行作用域规则校验。生命周期标注也对函数的实现者和调用者都产生了约束（这和类型系统类似）：

1. 对于函数的实现者：约束的是返回的引用的生命周期必须满足标注的生命周期，确保标注的生命周期可以和一个或多个参数的生命周期对齐，如标注了 `'a` 则不能返回 `'b`。
2. 对于函数的调用者：约束的是返回的引用的赋值给本地变量后，其在本地的使用范围必须小于传入引用参数的生命周期。

除此之外，通过显式标注生命周期，借用检查器在静态分析时可以确保不会出现悬垂引用或数据竞争，开发者能够清楚地看到引用的关系，这有助于提高代码的可读性和可维护性。

### 如何标注？

针对 Rust 只有一个返回值中只有一个生命周期标注的情况，可以对所有参数和返回值标注相同的生命周期参数。
采用这种方式进行标注，可以使函数体实现都可以通过借用检查器检查，编译通过。

例如：

```rs
fn f_xxx(a: &'z str, b: &'z str, c: &'z str) -> &'z str {}
```

不论函数体实现如何，无脑的将参数 `a` `b` `c` 中所有可能被返回的参数生命周期都标为 `'z` 以满足对于函数的实现者的约束，剩余参数的生命周期也都标为 `'a`，这样所有参数和返回值标注相同的生命周期参数了。

但是这样会增加对函数调用方的约束，因为返回引用的生命周期，将会缩小到参数中生命周期最小参数的那个参数。
导致函数调用时出现较大限制，无法满足更多的使用场景。

最佳实践：尽可能将各参数的生命周期标注成不同的。可以获得如下优势：

1. 更广泛的适用性：通过为每个参数使用不同的生命周期标注，函数可以返回任何一个参数的引用，而不必受到最短生命周期的限制。这使得函数在不同上下文中更具适用性。
2. 更清晰的意图：不同的生命周期标注可以更清晰地表达函数的意图，帮助调用者理解哪些参数的生命周期是相关的。

### 示例

#### 返回类型不是引用类型，无需标生命周期

```rs
fn get_0(_s1: &str, _s2: &str) -> i32 {
    0
}
```

标了也不会有任何问题

```rs
fn get_0<'a, 'b>(_s1: &'a str, _s2: &'b str) -> i32 {
    0
}
```

#### 编译报错，Rust 不允许函数返回局部变量的引用

```rs
fn get_local_ref() -> &'static str {
    let s = String::from("Rosie");
    &s
}
```

#### 函数参数标注相同的生命

```rs
fn max<'a, 'b>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

#### 报错的调用方式，根据标注 `result` 生命周期不能超过 `s2`

```rs
fn main() {
    let s1 = String::from("Lindsey");
    let result: &str;
    {
        let s2 = String::from("Rosie");

        result = max(&s1, &s2);

        println!("bigger one: {}", result);
    }

    println!("bigger one: {}", result);
}
```

成功的调用方式，保证 `result` 生命周期不能超过 `s2`

```rs
fn main() {
    let s1 = String::from("Lindsey");
    let result: &str;
    {
        let s2 = String::from("Rosie");

        result = max(&s1, &s2);
    
        println!("bigger one: {}", result);
    }
    // 不能在作用域之外使用 result
    // println!("bigger one: {}", result);
}
```

#### 参数 `s1` `s2` 标不同的生命周期，报错，因为 Rust 推断出函数实现不满足标注

```rs
fn max<'a, 'b>(s1: &'a str, s2: &'b str) -> &'a str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

只能返回生命周期是 `'a` 的引用，返回 `'b` 是不行的

#### 参数 `s1` `s2` 少标注一个生命周期，报错，因为 Rust 推断出函数实现不满足标注

```rs
fn max<'a, 'b>(s1: &'a str, s2: &str) -> &'a str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

只能返回生命周期是 `'a` 的引用。由于 `s2` 没有标注生命周期，编译器无法确定返回值是指向 `s1` (`'a`) 还是 `s2` (未知/匿名生命周期)，为了安全起见拒绝编译。

#### 以下示例可以标两个不同的生命周期

```rs
fn first_word<'a, 'b>(s: &'a String, _w: &'b String) -> &'a str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

fn main() {
    let s = String::from("Hello World.");
    let w = String::from("hahah");
    let fw = first_word(&s, &w);
    println!("fw: {}", fw);
}
```

或者，省略一个生命周期

```rs
fn first_word<'a>(s: &'a String, _w: &String) -> &'a str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

fn main() {
    let s = String::from("Hello World.");
    let w = String::from("hahah");
    let fw = first_word(&s, &w);
    println!("fw: {}", fw);
}
```

也可以标相同的生命周期

```rs
fn first_word<'a, 'b>(s: &'a String, _w: &'a String) -> &'a str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

fn main() {
    let s = String::from("Hello World.");
    let w = String::from("hahah");
    let fw = first_word(&s, &w);
    println!("fw: {}", fw);
}
```

标注相同生命周期，以下代码无法编译通过，函数使用范围受限

```rs
fn main() {
    let s = String::from("Hello World.");
    let fw: &str;
    {
        let w = String::from("hahah");
        fw = first_word(&s, &w);
    }
    println!("fw: {}", fw);
}
```

#### 通过元组返回多个值

```rs
fn longest_and_shortest<'a, 'b>(s1: &'a str, s2: &'b str) -> (&'a str, &'b str) {
    (s1, s2)
}

fn main() {
    let string1 = String::from("hello");
    let string2 = String::from("world!");

    let (longest, shortest) = longest_and_shortest(&string1, &string2);
    println!("Longest: {}, Shortest: {}", longest, shortest);
}
```

## 结构体引用类型属性的生命周期标注

## 泛型参数生命周期标注


---

<a href="https://github.com/liuyanjie/knowledge/tree/master/langs/rust/rust-lifecycle.md" >查看源文件</a>&nbsp;&nbsp;<a href="https://github.com/liuyanjie/knowledge/edit/master/langs/rust/rust-lifecycle.md">编辑源文件</a>

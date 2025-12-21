---
title: 深入理解 Rust async/.await 实现原理
date: 2025-12-21T11:00:00.000Z
updated: 2025-12-21T09:50:44.749Z
tags: [rust]
categories: [rust]
---



> 本文由AI辅助生成。本文内容基于 `学习 Rust async/.await 实现原理` 过程中与 `DeepSeek` 探讨相关问题的对话产生的内容，由 `DeepSeek` 整理，经 `Gemini/Claude` 审阅补充去 AI 味，最后经过人工审阅修改形成。

---

> Rust async/.await 已经是 Rust 高并发编程的最佳实践，深入理解 Rust async/.await 实现原理，有助于我们更好地使用 Rust 异步编程。
> 本文将从 Reactor 模式开始，深入探讨 Rust async/.await 实现原理。

## 引言

写 Rust 异步代码时，我们经常用到 `async/.await`。它让我们能用同步的思维方式写异步逻辑，非常舒服。但这种"魔法"背后到底发生了什么？

更令人疑惑的是：**为什么 Rust 需要显式引入 Executor（如 Tokio），而 Go 和 Node.js 似乎"没有"这个概念？**

这一切的核心在于 Rust 采用了 **编译时状态机** 配合 **Reactor 模式** 的设计，并将运行时组件剥离出了标准库。本文试图拆解这套机制，看看编译器到底帮我们做了什么。

## Reactor 模式：异步编程的基础模式

要理解 Rust 的异步模型，首先得聊聊 Reactor（反应器）模式。它是 Nginx、Node.js、Redis 等高性能服务背后的核心思想。简单来说，Reactor 就是通过**事件循环**和**非阻塞 I/O** 来解决高并发问题。

### 传统 Reactor 模式架构

```txt
┌─────────────────┐    ┌──────────────────┐
│   Event Loop    │◄───│   Demultiplexer  │
│                 │    │   (epoll/kqueue) │
└─────────────────┘    └──────────────────┘
         │                         ▲
         │ event dispatch          │ system events notification
         ▼                         │
┌─────────────────┐    ┌──────────────────┐
│  Event Handlers │    │   I/O Resources  │
│   (callback)    │    │   (sockets/files)│
└─────────────────┘    └──────────────────┘
```

**核心问题**：虽然 Reactor 性能很好，但手写回调（Callback）非常痛苦——也就是常说的 "回调地狱"。逻辑被打散，错误处理和状态管理变得异常复杂。

## Rust async/await 是如何工作的？

在深入细节之前，我们需要区分两种截然不同的 Future：

1. **组合型 Future (Async Block Future)**：由 `async fn` 或 `async` 块生成。它们负责**编排**异步逻辑，内部包含多个子 Future。
2. **叶子 Future (Leaf Future)**：通常由 Runtime 库（如 Tokio）提供的手写结构体（如 `TcpStream` 的 read/write）。它们处于调用树的末端，直接与 Reactor 交互。

### 1. 组合型 Future：编译器生成的状态机

当你在函数前加上 `async` 关键字，编译器其实会在编译阶段把它"重写"成一个匿名的**状态机**结构体。这个结构体实现了 `Future` trait。

这就好比编译器在帮我们手动写回调，但对外暴露的接口依然是清爽的同步风格。

```rust
// 开发者编写的代码
async fn simple_example(x: i32) -> i32 {
    let a = async_op1(x).await;
    let b = async_op2(a).await;
    b + 1
}

// 编译器生成的近似代码
struct SimpleExampleFuture {
    state: SimpleExampleState,
    x: i32,
}

enum SimpleExampleState {
    Start,
    AfterFirstAwait { op1_future: AsyncOp1Future },
    AfterSecondAwait { a: i32, op2_future: AsyncOp2Future },
    Done,
}
```

#### 状态机执行模型

每个 `.await` 点对应状态机的一个状态转换：

```rust
// 注意：这是简化的概念性伪代码，用于说明状态机原理
// 实际编译器生成的代码更复杂，且处理了 Pin、生命周期等细节
impl Future for SimpleExampleFuture {
    type Output = i32;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            match &mut self.state {
                SimpleExampleState::Start => {
                    // 初始化逻辑，只执行一次
                    let op1_future = async_op1(self.x);
                    self.state = SimpleExampleState::AfterFirstAwait { op1_future };
                    // 状态转换后继续循环，立即尝试 poll 新创建的 future
                }
                
                SimpleExampleState::AfterFirstAwait { ref mut op1_future } => {
                    // 注意：实际代码需要处理 Pin Projection（例如使用 pin-project 库）
                    // 才能从 Pin<&mut Self> 安全地获取字段的 Pin<&mut Field>
                    // 这里为简化演示使用 unsafe
                    match unsafe { Pin::new_unchecked(op1_future) }.poll(cx) {
                        Poll::Ready(a) => {
                            // 子 Future 完成，转换到下一状态
                            let op2_future = async_op2(a);
                            self.state = SimpleExampleState::AfterSecondAwait { a, op2_future };
                            // 继续循环处理新状态
                        }
                        Poll::Pending => return Poll::Pending, // 快速返回，不阻塞
                    }
                }
                
                SimpleExampleState::AfterSecondAwait { a, ref mut op2_future } => {
                    match unsafe { Pin::new_unchecked(op2_future) }.poll(cx) {
                        Poll::Ready(b) => {
                            let result = b + 1;
                            self.state = SimpleExampleState::Done;
                            return Poll::Ready(result);
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                
                SimpleExampleState::Done => {
                    panic!("Future polled after completion");
                }
            }
        }
    }
}
```

**关键理解**：

- 每个 `Await` 状态的核心是先执行 `poll`，再根据结果决定后续逻辑
- 在 `Poll::Pending` 时立即返回，不会执行任何依赖结果的代码
- 状态转换逻辑只在子 Future 完成时执行一次
- 这里的 `loop` 允许在一次 `poll` 调用中处理多个连续完成的状态转换，提高效率

### 2. 叶子 Future：手写的异步原语

叶子 Future 通常是手写的，它们不再包含子 Future，这类 Future 通常直接对接操作系统或运行时内核（如 I/O、定时器、通道等），通常不需要复杂的枚举状态，而是采用 **"Try-Check-Register"** 模式。

```rust
// 异步 TCP 读取的 Future（简化版叶子 Future）
// 注意：这是概念性示例，实际实现需要处理更多细节
struct AsyncReadFuture {
    socket: TcpStream,
    buffer: Vec<u8>,
    registered: bool,  // 标记是否已向 Reactor 注册
}

impl Future for AsyncReadFuture {
    type Output = Result<usize, io::Error>;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 1. 总是先尝试直接读取（非阻塞）
        match self.socket.try_read(&mut self.buffer) {
            Ok(n) => Poll::Ready(Ok(n)),
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                // 2. 如果返回 WouldBlock，说明数据未就绪
                // 只在未注册时才向 Reactor 注册，避免重复注册
                if !self.registered {
                    // 向 Reactor 注册感兴趣的事件（可读），并将当前 Waker 传递给 Reactor
                    // 当 socket 变得可读时，Reactor 会调用 waker.wake()
                    self.reactor().register(
                        self.socket.as_raw_fd(), 
                        Interest::READ, 
                        cx.waker().clone()
                    );
                    self.registered = true;
                } else {
                    // 已注册但 Waker 可能变化，需要更新
                    self.reactor().reregister(
                        self.socket.as_raw_fd(), 
                        Interest::READ, 
                        cx.waker().clone()
                    );
                }
                Poll::Pending
            }
            Err(e) => Poll::Ready(Err(e)),
        }
    }
}
```

**关键差异**：

- **组合型**：像导演，由编译器生成，关注流程控制（先做 A，再做 B）。通过 `enum` 记录当前执行到了哪一步。
- **叶子型**：像工人，由开发者编写，关注具体功能实现。

## Future 树与 Task 模型

### Future 的树状结构

异步调用会形成自然的树状结构：

```txt
Task (根Future)
├── async_fn1() 子Future
│   └── async_fn2() 孙Future
└── async_fn3() 子Future
    └── async_fn4() 孙Future
```

### Task：调度的基本单位

**Task** 是执行调度的基本单元，包含完整的 Future 树：

```rust
struct Task {
    future: Pin<Box<dyn Future<Output = ()> + Send>>, // 根Future
    id: TaskId,
    state: TaskState,
}

struct Executor {
    ready_queue: VecDeque<Task>,      // 就绪任务队列
    waiting_tasks: HashMap<TaskId, Task>, // 等待中的任务
}
```

## Waker：唤醒 Task

### Waker 与 Task 的关联

`Future` 被 `poll` 时如果还没准备好（返回 `Pending`），它必须告诉 Executor："等数据到了，记得叫醒我"。这就是 **Waker** 的作用。

Waker 机制的核心是将**具体事件**与**顶层任务（Task）**关联。当底层 I/O 就绪时，Waker 通知 Executor 将对应的 Task 重新加入就绪队列，Executor 随后再次 Poll 整个 Task（而不是只 Poll 某个子 Future）：

```rust
struct TaskWaker {
    task_id: TaskId,
    executor: Arc<Executor>,
}

impl Wake for TaskWaker {
    fn wake(self: Arc<Self>) {
        // 唤醒的是整个 Task，而不是某个具体的 Future
        self.executor.wake_task(self.task_id);
    }
}

impl Executor {
    fn wake_task(&self, task_id: TaskId) {
        // 将对应的 Task 重新放入就绪队列
        if let Some(task) = self.waiting_tasks.remove(&task_id) {
            self.ready_queue.push_back(task);
        }
    }
}
```

### 执行流程

1. **Executor 调度**：从就绪队列取出 Task，调用其根 Future 的 `poll()`
2. **递归 Polling**：根 Future 的 `poll()` 会递归调用子 Future 的 `poll()`
3. **等待注册**：如果叶子 Future 未就绪，它会存储 Waker 并返回 `Pending`
4. **事件触发**：异步操作完成时调用 Waker 的 `wake()` 方法
5. **重新调度**：Waker 将对应的 Task 重新放入就绪队列

## Executor 与 Reactor 协作模型

### 整体架构：三层协作模型

```txt
┌────────────────────────────────────────────────────┐
│                   Executor                         │
│    ┌─────────────────────────────────────────┐     │
│    │             Future (StateMachine)       │     │
│    │  ┌─────────────┐ ┌─────────────┐        │     │
│    │  │  SubFuture  │ │  SubFuture  │ ...    │     │
│    │  └─────────────┘ └─────────────┘        │     │
│    └─────────────────────────────────────────┘     │
└────────────────────────────────────────────────────┘
                          │
┌────────────────────────────────────────────────────┐
│                   Reactor                          │
│    ┌────────────────────────────────────────────┐  │
│    │       Event Multiplexing (epoll)           │  │
│    │  ┌─────┐ ┌─────┐ ┌───────┐ ┌────────┐      │  │
│    │  │ I/O │ │ I/O │ │ Timer │ │ Signal │ ...  │  │
│    │  └─────┘ └─────┘ └───────┘ └────────┘      │  │
│    └────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

## 核心组件深度解析

### 1. Reactor：事件驱动的引擎

Reactor 负责监听所有异步 I/O 事件，是整个系统的感知器官。在实际的 Tokio 实现中，Reactor 通常与 Executor 集成在同一个事件循环中运行：

```rust
// Reactor 核心职责（简化示例）
struct Reactor {
    poll: mio::Poll,           // 系统事件多路复用
    events: mio::Events,       // 就绪事件集合
    waker_map: HashMap<Token, Waker>, // Token 到 Waker 的映射
}

impl Reactor {
    // 注意：在真实的 Tokio 中，这个方法通常不是独立循环
    // 而是集成在 Executor 的调度循环中
    fn poll_events(&mut self, timeout: Option<Duration>) {
        // 等待事件就绪（可能阻塞）
        self.poll.poll(&mut self.events, timeout).unwrap();
        
        for event in &self.events {
            let token = event.token();
            
            // 查找对应的 Waker 并唤醒
            // 注意：remove 后如果事件再次触发，需要 Future 重新注册
            if let Some(waker) = self.waker_map.remove(&token) {
                waker.wake();
            }
        }
    }
}
```

**关键要点**：

- **Reactor 与 Executor 的协作**：在单线程运行时中，Reactor 和 Executor 通常在同一个线程交替运行
- **事件注册的生命周期**：Waker 被触发后从 map 中移除，需要 Future 在下次 poll 时重新注册
- **超时控制**：可以设置超时避免无限期阻塞，让 Executor 有机会处理其他任务

### 2. Waker：连接 Reactor 与 Executor 的桥梁

Waker 是 Reactor 模式中回调机制的现代化身。它本质上是一个引用计数的智能指针，可以安全地跨线程传递和克隆：

```rust
struct TaskWaker {
    task_id: TaskId,
    executor: Arc<Executor>,
}

impl Wake for TaskWaker {
    fn wake(self: Arc<Self>) {
        // 将 Task 重新加入调度队列
        // 注意：Waker 不直接持有 Reactor 引用
        // Reactor 注册逻辑在 Future::poll 中完成
        self.executor.schedule(self.task_id);
    }
}
```

**Waker 的关键特性**：

- **可克隆**：`cx.waker().clone()` 创建新的引用，不会复制底层数据
- **线程安全**：可以在不同线程间传递（实现了 `Send` 和 `Sync`）
- **引用计数**：底层使用 `Arc`，自动管理生命周期

### 3. Future 与 Reactor 的交互

前文提到的 **叶子 Future** 就是 Reactor 事件处理逻辑的载体。它连接了高级的 `await` 语法和底层的事件通知机制。

## 实际应用：Tokio 运行时架构

Tokio 是 Rust 最流行的异步运行时，完美体现了这种架构：

```rust
#[tokio::main]
async fn main() {
    // 背后启动完整的 Reactor + Executor 基础设施
    let response = reqwest::get("https://example.com").await.unwrap();
    println!("Status: {}", response.status());
}
```

**Tokio 组件分解**：

- **Reactor**: 基于 mio 的事件循环
- **Executor**: 工作窃取线程池
- **Timer**: 异步定时器组件
- **I/O Driver**: 统一 I/O 事件处理

## 与 Go/Node.js 的对比

为什么 Rust 需要我们显式引入 Executor（如 Tokio），而 Go 和 Node.js 似乎"没有"这个概念？

* **Go/Node.js**：Runtime 内置了复杂的调度器（Executor）、事件循环（Reactor）甚至垃圾回收（GC）。
  * **优点**：开箱即用，开发体验统一。
  * **缺点**：运行时体积大，难以裁剪。在内存受限的环境中，很可能无法使用或消耗太多资源。
  * **看起来"没有"Executor**：其实是**内置**了且不可替换，用户感知不到。

* **Rust**：标准库只定义了 `Future` **接口**（Trait），不包含具体的**实现**（Executor/Reactor）。
  * **优点**：不绑定任何并发编程模型。不需要异步时就没有运行时开销；在嵌入式环境可以用极简的 Executor；在服务器环境可以用更复杂的 Tokio。
  * **缺点**：需要感知 Executor 增加理解学习成本。需要显式编写相关代码。
  * **显式引入 Executor**：因为标准库没提供，所以必须引入第三方实现（如 Tokio）。

## 总结

Rust 的异步模型其实就是把 **Reactor 模式** 和 **编译时状态机** 结合在了一起：

1. **写起来像同步**：`async/.await` 语法让我们不用写回调。
2. **跑起来是状态机**：编译器把代码转换成高效的状态机。
3. **底层还是 Reactor**：依靠 epoll/kqueue 等系统调用来驱动事件。

理解这个架构，能帮你更好地理解为什么 Rust 的 Future 需要 `poll`，为什么需要 `Executor`，以及为什么在 Rust 里你甚至可以自己写一个 Runtime。

## 补充说明

### 关于代码示例

本文中的代码示例都是**简化的概念性代码**，目的是帮助理解核心机制。实际生产代码需要处理更多细节：

- **Pin 的处理**：真实代码需要正确处理 `Pin` 投影，通常使用 `pin-project` 等库
- **错误处理**：需要妥善处理各种错误情况
- **性能优化**：真实 Runtime 会有更多优化，如批量处理、工作窃取等
- **并发安全**：多线程环境下需要考虑同步问题

### Pin 为什么必要？

`Future` 需要 `Pin` 的核心原因是**自引用结构**：

```rust
// async fn 生成的状态机可能包含自引用
struct SelfReferentialFuture {
    data: String,
    ptr: *const String,  // 指向 self.data
}
```

如果这个结构体被移动到新的内存地址，`ptr` 就会失效。`Pin` 保证了被 pin 住的值不会再被移动，使得自引用结构安全可用。

### Executor 的类型

不同场景需要不同的 Executor：

- **单线程 Executor**：适合嵌入式或简单场景，开销小
- **多线程 Executor**：如 Tokio 的工作窃取线程池，适合高并发服务器
- **自定义 Executor**：可以根据特定需求定制调度策略

Rust 的设计允许你根据实际需求选择最合适的 Runtime，而不是被强制使用统一的实现。


---

<a href="https://github.com/liuyanjie/knowledge/tree/master/langs/rust/rust-asyncawait.md" >查看源文件</a>&nbsp;&nbsp;<a href="https://github.com/liuyanjie/knowledge/edit/master/langs/rust/rust-asyncawait.md">编辑源文件</a>

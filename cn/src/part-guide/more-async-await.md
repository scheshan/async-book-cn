# 更多 async/await 主题

## 单元测试

如何对异步代码做单元测试？问题在于只能在异步上下文中 `await`，而 Rust 的单元测试本身不是 async 的。好在多数运行时会提供类似 `async main` 的测试便捷属性。使用 Tokio 时如下：

```rust,norun
#[tokio::test]
async fn test_something() {
  // Write a test here, including all the `await`s you like.
}
```

测试有多种配置方式，详见[文档](https://docs.rs/tokio/latest/tokio/attr.test.html)。

异步代码测试还有一些更高级主题（例如竞态、死锁等），本指南[后面]()会部分涉及。


## 阻塞与取消

用 async Rust 编程时，阻塞与取消很重要。它们不局限于某个特性或函数，而是贯穿整个系统的性质，必须理解才能写出正确代码。

### 阻塞 I/O

当线程（这里指 OS 线程，不是异步任务）无法推进时，我们说它被阻塞了。通常是因为在等 OS 代它完成任务（多半是 I/O）。重要的是，线程阻塞时 OS 知道不要调度它，好让其他线程推进。在多线程程序里这没问题——阻塞线程等待时其他线程仍能工作。但在异步程序里，同一 OS 线程上还应调度其他任务，而 OS 不知道这些任务，会让整条线程一直等。于是不是只有单个任务在等自己的 I/O（这可以接受），而是许多任务都得等（这不行）。

非阻塞/异步 I/O 很快会讲到；现在只需知道：非阻塞 I/O 是异步运行时*知晓*的 I/O，因此只有当前任务在等待，线程本身不会被阻塞。在异步任务里*必须*只用非阻塞 I/O，绝不要用阻塞 I/O（Rust 标准库只提供阻塞 I/O）。

### 阻塞计算

你也可以通过计算阻塞线程（与阻塞 I/O 不完全相同，因为 OS 不参与，但效果类似）。若有长时间计算（无论是否含阻塞 I/O）却不把控制权交给运行时，该任务就永远不会给调度器机会去调度其他任务。记住异步编程使用协作式多任务——这里任务不协作，其他任务就得不到执行机会。缓解办法后面会讨论。

还有许多其他方式会阻塞整条线程，本指南会多次回到阻塞话题。

### 取消

取消指停止 Future（或任务）的执行。在 Rust 中（与许多其他 async/await 系统不同），Future 必须由外部力量（如异步运行时）驱动；若不再被驱动，就不会再执行。若 Future 被 drop（记住 Future 只是普通 Rust 对象），它就无法再推进，即被取消。

取消可以通过几种方式发起：

- 直接 drop 你拥有的 Future。
- 对任务的 `JoinHandle`（或 `AbortHandle`）调用 [`abort`](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html#method.abort)。
- 通过 [`CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html)（需要被取消的 Future 注意到 token 并协作式自行取消）。
- 由 [`select`](https://docs.rs/tokio/latest/tokio/macro.select.html) 等函数或宏隐式触发。

中间两种是 Tokio 特有的，但多数运行时有类似设施。使用 `CancellationToken` 需要被取消的 Future 配合，其他方式则不需要。在这些情况下，被取消的 Future 不会收到取消通知，也没有清理机会（除析构函数外）。即使 Future 带有 cancellation token，仍可能通过其他方式被取消，且不会触发该 token。

从编写异步代码的视角（async 函数、块、Future 等），代码可能在任意 `await`（包括宏里的隐藏 `await`）处停止执行且永不继续。要使代码正确（特别是*取消安全*），无论正常完成还是在任意 await 点终止都必须行为正确[^cfThreads]。

```rust,norun
async fn some_function(input: Option<Input>) {
    let Some(input) = input else {
        return;           // Might terminate here (`return`).
    };

    let x = foo(input)?;  // Might terminate here (`?`).

    let y = bar(x).await; // Might terminate here (`await`).

    // ...

    //                       Might terminate here (implicit return).
}
```

一个会出错的例子：异步函数把数据读入内部缓冲区，再 await 下一条数据。若读取是破坏性的（无法从原处重读）且函数被取消，内部缓冲区会被 drop，其中数据丢失。必须考虑取消 Future、重启 Future，或启动另一个会触及相同数据的新 Future 时，Future 及其触及的数据会受到什么影响。

本指南会多次回到取消与取消安全；参考部分还有一整章[专论该主题]()。

[^cfThreads]: 有趣的是，可以把异步中的取消与取消线程对比。取消线程是可能的（例如 C 里用 `pthread_cancel`，Rust 没有直接等价 API），但几乎总是极差的主意，因为被取消的线程可能在任意位置终止。相比之下，取消异步任务只能在 await 点发生。因此很少在不终止整个进程的情况下取消 OS 线程，程序员通常不必担心这种事。但在 async Rust 中，取消*确实*可能发生，我们会逐步讨论如何应对。

## 异步块

普通块（`{ ... }`）在源码里把代码分组，并为名称创建封装作用域。运行时块按顺序执行，求值为最后一条表达式的值（若无尾部表达式则为单元类型 `()`）。

与异步函数类似，**异步块**是普通块的延后版本。异步块在源码里同样分组代码与名称，但运行时不会立即执行，求值结果是 Future。要执行块并得到结果，必须 `await`。例如：

```rust,norun
let s1 = {
    let a = 42;
    format!("The answer is {a}")
};

let s2 = async {
    let q = question().await;
    format!("The question is {q}")
};
```

若执行这段代码，`s1` 是可打印的字符串，而 `s2` 是 Future；`question()` 尚未被调用。要打印 `s2`，须先 `s2.await`。

异步块是开启异步上下文、创建 Future 的最简单方式，常用于只在一处使用的小 Future。

遗憾的是，异步块的控制流有点别扭。因为异步块创建的是 Future 而非直接执行，在控制流上更像函数而不是普通块。`break` 和 `continue` 不能像穿过普通块那样穿过异步块，须改用 `return`：

```rust,norun
loop {
    {
        if ... {
            // ok
            continue;
        }
    }

    async {
        if ... {
            // not ok
            // continue;

            // ok - continues with the next execution of the `loop`, though note that if there was
            // code in the loop after the async block that would be executed.
            return;
        }
    }.await
}
```

要实现 `break`，需要检查块的值（常见写法是用 [`ControlFlow`](https://doc.rust-lang.org/std/ops/enum.ControlFlow.html) 作为块的值，这样也能用 `?`）。

同样，异步块内的 `?` 在出错时会终止该 Future 的执行，使被 `await` 的块得到错误值，但*不会*退出外层函数（普通块里的 `?` 会）。需要在 `await` 之后再跟一个 `?`：

```rust,norun
async {
    let x = foo()?;   // This `?` only exits the async block, not the surrounding function.
    consume(x);
    Ok(())
}.await?
```

这常让编译器困惑，因为（与函数不同）异步块的「返回」类型没有显式写出。可能需要在变量上加类型标注或用 turbofish，例如上例用 `Ok::<_, MyError>(())` 而不是 `Ok(())`。

返回异步块的函数与 async 函数非常接近。写 `async fn foo() -> ... { ... }` 大致等价于 `fn foo() -> ... { async { ... } }`。事实上，从调用方视角两者等价，在两者之间切换不是破坏性变更。实现 async trait 时也可以用一种覆盖另一种（见下文）。但异步块版本需要把类型写清楚，把 `Future` 显式标出：`async fn foo() -> Foo` 变成 `fn foo() -> impl Future<Output = Foo>`（可能还要显式写出其他约束，如 `Send` 和 `'static`）。

通常更倾向 async 函数形式，更简单清晰。但异步块版本更灵活：可以在函数被*调用*时执行块外代码，在结果被*await* 时执行块内代码。


## 异步闭包

- 闭包
  - 即将支持（https://github.com/rust-lang/rust/pull/132706，https://blog.rust-lang.org/inside-rust/2024/08/09/async-closures-call-for-testing.html）
  - 闭包中的异步块 vs 异步闭包


## 生命周期与借用

- 上文提到的 `'static` 生命周期
- Future 上的生命周期约束（`Future + '_` 等）
- 跨 await 点的借用
- 肯定还有更多与 async 函数相关的生命周期问题……


## Future 上的 `Send + 'static` 约束

- 为何存在、多线程运行时
- 用 `spawn_local` 规避
- 什么使 async fn 成为 `Send + 'static`，以及如何修复相关 bug


## 异步 trait

- 语法
  - `Send + 'static` 问题及规避
    - trait_variant
    - 显式 Future
    - 返回类型记号（https://blog.rust-lang.org/inside-rust/2024/09/26/rtn-call-for-testing.html）
- 覆盖
  - 方法的 future 写法 vs async 写法
- 对象安全
- 捕获规则（https://blog.rust-lang.org/2024/09/05/impl-trait-capture-rules.html）
- 历史与 async-trait crate


## 递归

- 现已允许（相对较新），但需要显式装箱
  - 对 Future 的前向引用、Pin
  - https://rust-lang.github.io/async-book/07_workarounds/04_recursion.html
  - https://blog.rust-lang.org/2024/03/21/Rust-1.77.0.html#support-for-recursion-in-async-fn
  - async-recursion 宏（https://docs.rs/async-recursion/latest/async_recursion/）

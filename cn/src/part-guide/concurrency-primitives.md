# 并发的组合 Future

本章介绍更多组合 Future 的方式，尤其是让 Future **并发**执行（但**不并行**）的新手段。表面上，本章引入的函数/宏很简单，底层概念却可能很微妙。我们会先回顾 Future、并发与并行，也建议你重读较早一节中[并发与并行](concurrency.md#并发与并行)的对比。

Future 是延后计算。用 `await` 可推进 Future，把控制权交给运行时，使当前任务等待计算结果。若 `a`、`b` 是 Future，可顺序组合（先完成 `a` 再完成 `b`）：`async { a.await; b.await }`。

我们也见过用 `spawn` **并行**组合 Future：`async { let a = spawn(a); let b = spawn(b); (a.await, b.await) }` 让两个 Future 并行运行。注意元组里的 `await` 不是在 await Future 本身，而是在 await `JoinHandle`，以在 Future 完成时取得结果。

本章介绍两种**并发但不并行**地组合 Future 的方式：`join` 与 `select`/`race`。两种情况下，Future 通过时间片轮转并发执行：每个被组合的 Future 轮流执行一轮，再轮到下一个。这*不经过异步运行时*（因此没有多条 OS 线程，也没有并行可能）。组合构造在本地交错执行这些 Future。可以把它们想成在单个异步任务内执行其组件 Future 的迷你执行器。

`join` 与 `select`/`race` 的根本区别在于 Future 完成时如何处理：**join** 在所有 Future 都完成时结束；**select/race** 在有一个 Future 完成时结束（其余被取消）。两者也都有处理错误的变体。

这些构造（或类似概念）常与 stream 一起使用，下文会略提，更多见 [stream 一章](streams.md)。

若需要并行（或你并未明确不要并行），生成任务往往比这些组合构造更简单：通常更少出错、更通用、性能更可预测。另一方面，spawn 天生更缺乏[结构化](../part-reference/structured.md)，生命周期与资源管理更难推理。

值得稍深入考虑性能问题。并发组合的潜在问题是时间片分配的**公平性**。若程序里有 100 个任务，通常最优是让每个任务约占 1% 处理器时间（若都在等待，则唤醒机会相当）。spawn 100 个任务时大致如此。但若 spawn 两个任务，并在其中一个上 `join` 99 个 Future，调度器只知道两个任务，一个占 50%，99 个 Future 各约 0.5%。

任务分布通常不会这么偏，且 `join`/`select` 等常用于超时等场景，这种行为反而合意。但仍值得考虑，确保程序具备你期望的性能特征。


## join

Tokio 的 [`join` 宏](https://docs.rs/tokio/latest/tokio/macro.join.html) 接受一组 Future，让它们全部并发执行到完成（把所有结果作为元组返回）。所有 Future 都完成时返回。Future 始终在同一线程上执行（并发、非并行）。

简单例子：

```rust,norun
async fn main() {
  let (result_1, result_2) = join!(do_a_thing(), do_a_thing());
  // Use `result_1` and `result_2`.
}
```

这里两次 `do_a_thing` 并发执行，两者都完成后结果才就绪。注意不必再 `await` 取结果——`join!` 隐式 await 其 Future 并产生值。它不创建 Future，但仍须在异步上下文中使用（例如在 async 函数内）。

上例看不出来，`join!` 接受求值为 Future 的表达式[^into]。`join` 不在其体内创建异步上下文，也不应对传入 `join` 的 Future 先 `await`（否则会在被 join 的 Future 执行前就求值完毕）。

因所有 Future 在同一线程执行，任一 Future 阻塞线程则全部无法推进。若使用 mutex 或其他锁，一个 Future 等另一个持有的锁时很容易死锁。

[`join`](https://docs.rs/tokio/latest/tokio/macro.join.html) 不关心 Future 的结果类型本身。若某 Future 被取消或返回错误，不影响其他 Future——它们继续执行。若要「快速失败」，用 [`try_join`](https://docs.rs/tokio/latest/tokio/macro.try_join.html)。`try_join` 与 `join` 类似，但若任一 Future 返回 `Err`，其余 Future 会被取消，`try_join` 立即返回该错误。

在较早的 [async/await](async-await.md) 一章里，我们用「join」指合并已 spawn 的任务。顾名思义，合并 Future 与合并任务相关：并发执行多个 Future 并等待结果后再继续。语法不同：用 `JoinHandle` 还是 `join` 宏，但思想相近。关键区别：合并**任务**时任务并发且并行；用 `join!` 时 Future 并发但不并行。此外，spawn 的任务 panic 由运行时捕获；`join` 里某 Future panic 则整个任务 panic。spawn 的任务由运行时调度器调度；`join!` 则在本地「调度」（同一任务、宏执行的时间范围内）。


### 替代方案

并发运行 Future 并收集结果是常见需求。除非有充分理由（即你明确不要并行，即便如此也可能更倾向 [`spawn_local`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_local.html)），否则应优先用 `spawn` 与 `JoinHandle`。 [`JoinSet`](https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html) 以类似 `join!` 的方式管理这类已 spawn 任务。

多数运行时（以及 [futures.rs](https://docs.rs/futures/latest/futures/macro.join.html)）有与 Tokio `join` 宏等价的设施，行为大体相同。也有 `join` **函数**，比宏略不灵活。例如 futures.rs 有 [`join`](https://docs.rs/futures/latest/futures/future/fn.join.html)（两个 Future）、[`join3`](https://docs.rs/futures/latest/futures/future/fn.join3.html) 至 [`join5`](https://docs.rs/futures/latest/futures/future/fn.join5.html)，以及 [`join_all`](https://docs.rs/futures/latest/futures/future/fn.join_all.html)（集合），各有 `try_` 变体。

[futures-concurrency](https://docs.rs/futures-concurrency/latest) 也提供 join（与 try_join）。在其风格下，这些操作是 Future 组（元组、`Vec`、数组等）上的 trait 方法。例如合并两个 Future：`(fut1, fut2).join().await`（这里 `await` 是显式的）。

若要 join 的 Future 集合在运行时动态变化（例如随网络输入不断创建新 Future），或希望在各 Future 完成时逐个取结果而非等全部完成，则需用 stream 以及 [`FuturesUnordered`](https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html) 或 [`FuturesOrdered`](https://docs.rs/futures/latest/futures/stream/struct.FuturesOrdered.html)，在 [stream](streams.md) 一章介绍。


[^into]: 表达式类型须实现 `IntoFuture`。宏会求值表达式并转换为 Future，即不必直接是 Future，而是可转换为 Future 的类型——差别很小。表达式本身会在任何结果 Future 执行*之前*按顺序求值。


## 竞速 / select

与 join 相对的是 **race**（即 **select**）。race/select 下 Future 并发执行，但只等*第一个*完成，然后取消其余。听起来与 join 相似，但因涉及取消而复杂得多（有时更易出错）。

用 Tokio 的 [`select`](https://docs.rs/tokio/latest/tokio/macro.select.html) 宏的例子：

```rust,norun
async fn main() {
  select! {
    result = do_a_thing() => {
      println!("computation completed and returned {result}");
    }
    _ = timeout() => {
      println!("computation timed-out");
    }
  }
}
```

比 `join` 宏更有趣：`select` 有点像 `match`，但所有分支并发运行，最先完成的分支以其结果执行体（其余分支不执行，Future 通过 `drop` 取消）。上例中 `do_a_thing` 与 `timeout` 并发，只有一个 `println` 会执行。与 `join` 一样，await 是隐式的。

Tokio 的 `select` 宏支持多种特性：

- **模式匹配**：每分支 `=` 左侧可以是模式，仅当 Future 结果匹配时才执行该分支体；不匹配则不再 poll 该 Future（其他 Future 仍 poll）。对可选返回值有用，例如 `Some(x) = do_a_thing() => { ... }`。
- **`if` 守卫**：每分支可有 `if` 守卫。`select` 运行时，表达式求值为 Future 后评估守卫，仅守卫为真才 poll 该 Future。例如 `x = do_a_thing() if false => { ... }` 永不被 poll。注意守卫只在宏初始化时评估，poll 过程中不重新评估。
- **`else` 分支**：可有 `else => { ... }`，当所有 Future 都停止且没有任何分支体被执行时运行。若无 `else` 且发生这种情况，`select` 会 panic。

`select!` 宏的值是所执行分支的值（同 `match`），故所有分支类型须相同。若要在 `select` 外使用结果，可写成：

```rust,norun
async fn main() {
  let result = select! {
    result = do_a_thing() => {
      Some(result)
    }
    _ = timeout() => {
      None
    }
  };

  // Use `result`
}
```

与 `join!` 一样，`select!` 不以特殊方式处理 `Result`（除前述模式匹配外）；若某分支以错误完成，其余分支会被取消，错误作为 select 的结果（与成功完成相同方式）。

`select` 宏内在使用取消，若程序要避免取消，就必须避免 `select!`。事实上 `select` 常常是异步程序取消的主要来源。如[别处](../part-reference/cancellation.md)所述，取消有许多微妙问题可导致 bug。尤其注意：`select` 通过直接 drop Future 来取消，不会通知被 drop 的 Future，也不会触发 cancellation token 等。

`select!` 常在循环中处理 stream 或其他 Future 序列，又多一层复杂与 bug 机会。若每轮循环创建新的独立 Future，情况尚简单，但这很少够用。通常需在迭代间保留状态。常见用法是在循环里用 `select` 处理 stream 的每一项，例如：

```rust,norun
async fn main() {
  let mut stream = ...;

  loop {
    select! {
      result = stream.next() => {
        match result {
          Some(x) => println!("received: {x}"),
          None => break,
        }
      }
      _ = timeout() => {
        println!("time out!");
        break;
      }
    }
  }
}
```

从 `stream` 读取并打印，直到没有数据或等待超时。超时情况下 stream 中剩余数据如何处理取决于 stream 实现（可能丢失或重复！）。这是取消时行为为何重要（且棘手）的一例。

我们可能想在多次迭代间复用 Future，而不只是 stream——例如与超时 Future 竞速，且超时适用于所有迭代而非每轮新建超时。可在循环外创建 Future 并引用它：

```rust,norun
async fn main() {
  let mut stream = ...;
  let mut timeout = timeout();

  loop {
    select! {
      result = stream.next() => {
        match result {
          Some(x) => println!("received: {x}"),
          None => break,
        }
      }
      // Create a reference to `timeout` rather than moving it.
      _ = &mut timeout => {
        println!("time out!");
        break;
      }
    }
  }
}
```

在循环里对 `select!` 使用循环外创建的 Future 或 stream 时，有几个重要细节——这是 `select` 工作方式的必然结果。以下用上一例的 `timeout` 逐步说明：

- `timeout` 在循环外创建并开始倒计时。
- 每轮循环 `select` 创建对 `timeout` 的引用，但不改变其状态。
- `select` 执行时 poll `timeout`：尚有时间则返回 `Pending`，时间到则返回 `Ready` 并执行其分支。

上例中 `timeout` 就绪时我们 `break`。若不 `break` 呢？`select` 会再次 poll `timeout`，而 `Future` [文档](https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll) 说不应如此！`select` 无法在迭代间保存状态来判断是否还应 poll `timeout`。取决于 `timeout` 的实现，可能 panic、逻辑错误或崩溃。

可用几种方式避免这类 bug：

- 使用[融合（fused）](futures.md#fusing)的 [future](https://docs.rs/futures/latest/futures/future/trait.FutureExt.html#method.fuse) 或 [stream](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.fuse)，使重复 poll 安全。
- 调整结构，确保 Future 不会被重复 poll，例如 `break` 出循环（如上），或使用 `if` 守卫。

`&mut timeout` 的类型呢？假设 `timeout()` 返回实现 `Future` 的类型，可能是 async 函数的匿名类型，也可能是 `Timeout` 等具名类型。假设后者以便举例（逻辑对匿名类型同样适用）。`Timeout` 实现了 `Future`，`&mut Timeout` 是否也实现 `Future`？不一定！有[空白实现](https://doc.rust-lang.org/std/future/trait.Future.html#impl-Future-for-%26mut+F) 成立，但仅当 `Timeout` 实现 `Unpin`。并非所有 Future 都 `Unpin`，因此上例类代码常报类型错误，用 `pin` 宏即可修复，例如 `let mut timeout = pin!(timeout());`

在循环里用 `select` 取消是微妙 bug 的富矿，常发生在 Future 持有涉及某些数据的状态（而非数据本身）时：取消 drop Future 会丢掉状态，底层数据却未更新，导致数据丢失或重复处理。


### 替代方案

futures.rs 有自己的 [`select` 宏](https://docs.rs/futures/latest/futures/macro.select.html)，futures-concurrency 有 [Race trait](https://docs.rs/futures-concurrency/latest/futures_concurrency/future/trait.Race.html)，都是 Tokio `select` 的替代。核心语义相同：并发竞速多个 Future，处理最先完成者的结果并取消其余，但语法与细节不同。

futures.rs 的 `select` 表面上类似 Tokio，差异概括如下：

- Future 必须始终为 fused（类型系统强制）。
- `select` 有 `default` 与 `complete` 分支，而非 `else`。
- `select` 不支持 `if` 守卫。

futures-concurrency 的 `Race` 语法很不同，类似其 `join` 风格，例如 `(future_a, future_b).race().await`（也支持 `Vec` 与数组）。语法不如宏灵活，但与多数 async 代码契合。注意在循环里用 `race` 仍可能有与 `select` 相同的问题。

与 `join` 一样，spawn 任务让其并行执行常是 `select` 的好替代。但要在第一个完成后取消其余任务需额外工作，可用 channel 或 cancellation token。无论哪种，被取消的任务都需配合行动，从而可做清理或优雅关闭。

`select`（尤其在循环内）常用于 stream。有些 stream 组合子可替代部分 `select` 用法，例如 futures-concurrency 的 [`merge`](https://docs.rs/futures-concurrency/latest/futures_concurrency/stream/trait.Merge.html) 适合合并多个 stream。


## 结语

本节讨论了两种让一组 Future 并发运行的方式：**join** 等待全部完成；**select**（竞速）等待第一个完成。与 spawn 任务不同，这些组合不使用并行。

`join` 与 `select` 作用于事先已知的一组 Future（常在写程序时而非运行时）。有时要被组合的 Future 事先未知——须在执行过程中动态加入。这需要 [stream](streams.md) 及其组合操作。

值得再次强调：尽管这些组合算子强大且富有表现力，使用任务与 spawn 往往更简单、更合适：并行通常更可取，更少遇到取消或阻塞相关的 bug，资源分配通常更公平（或至少更简单）、更可预测。

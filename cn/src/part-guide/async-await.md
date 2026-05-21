# async 与 await

本章开始用 Rust 做异步编程，并介绍 `async` 与 `await` 关键字。

`async` 是函数（以及 trait 等其他项，后面会讲到）上的标注；`await` 是用于表达式的运算符。在深入这两个关键字之前，需要先了解 Rust 异步编程的几个核心概念——这承接上一章的讨论，这里我们会直接与 Rust 编程对应起来。

## Rust 异步概念

### 运行时

异步任务必须被管理和调度。通常任务数多于可用核心，因此不能全部同时运行。当一个任务停止执行时，必须选出另一个来执行。若任务在等待 I/O 或其他事件，就不应被调度；事件完成后，又应被调度。这需要与 OS 交互并管理 I/O 工作。

许多编程语言提供运行时。常见情况下，运行时远不止管理异步任务——还可能管理内存（包括垃圾回收）、参与异常处理、为 OS 提供抽象层，甚至是一个完整的虚拟机。Rust 是底层语言，力求最小的运行时开销，因此异步运行时的职责范围比许多语言小得多。异步运行时的设计与实现方式也很多，Rust 让你按需求选择，而不是内置唯一一种。这也意味着入门异步编程需要多一步。

除了运行和调度任务，运行时还必须与 OS 交互以管理异步 I/O，并向任务提供定时器功能（与 I/O 管理有交集）。对运行时应如何组织没有硬性规则，但一些术语与职责划分很常见：

- *reactor*、*event loop* 或 *driver*（同义）：分发 I/O 与定时器事件，与 OS 交互，在最底层推进执行；
- *scheduler*（调度器）：决定任务何时、在哪个 OS 线程上执行；
- *executor* 或 *runtime*：把 reactor 与 scheduler 组合起来，是面向用户、用于运行异步任务的 API；*runtime* 也用来指整个功能库（例如 Tokio crate 中的全部内容，而不只是由 [`Runtime`](https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html) 类型表示的 Tokio 执行器）。

除上述执行器外，运行时 crate 通常还包含许多工具 trait 与函数，例如用于 I/O 的 trait（如 `AsyncRead`）及实现、常见 I/O 任务（网络、文件系统等）、锁、通道与其他同步原语、定时工具、与 OS 协作的工具（如信号处理）、处理 Future 与 stream（异步迭代器）的工具函数，以及监控与观测工具。本指南会涵盖其中许多内容。

可选的异步运行时很多，调度策略差异很大，或针对特定任务/领域优化。本指南大部分内容使用 [Tokio](https://tokio.rs/) 运行时——通用、生态中最流行，适合入门与生产。某些场景下换用其他运行时可能性能更好或代码更简单。后面会讨论其他运行时及如何取舍，甚至如何自己实现。

要尽快跑起来，只需少量样板代码。在 `Cargo.toml` 中加入 Tokio 依赖（与其他 crate 一样）：

```
[dependencies]
tokio = { version = "1", features = ["full"] }
```

并在 `main` 上使用 `tokio::main` 标注，使 `main` 可以是异步函数（否则 Rust 不允许）：

```rust,norun
#[tokio::main]
async fn main() { ... }
```

就这样！可以开始写异步代码了。

`#[tokio::main]` 会初始化 Tokio 运行时，并启动一个异步任务来运行 `main` 中的代码。本指南后面会更详细说明该标注在做什么，以及如何不用它来写异步代码（从而更灵活）。

### futures-rs 与生态

TODO：背景与历史、futures-rs 的用途——曾广泛使用，现在多半不需要；与 Tokio 及其他运行时的重叠（有时语义细微不同）、何时仍可能需要（直接操作 Future，尤其是自己实现 Future、stream、一些工具）。

其他生态：Yosh 的 crate、替代运行时、实验性项目等？

### Future 与任务

Rust 中异步并发的基本单位是 **Future**。Future 只是实现了 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的普通 Rust 对象（通常是 struct 或 enum）。Future 表示一项*延后*的计算——即在将来某个时刻才会就绪的计算。

本指南会大量讨论 Future，但入门时不必过分纠结，接下来几节会多次提到，直到后面才真正定义并直接使用。Future 的一个重要方面是可以组合成更大、更「复杂」的 Future（*如何*组合后面会详述）。

上一章和本章里，我非正式地用过「异步任务」一词，指一段逻辑上的执行序列——类似线程，但在程序内管理而非由 OS 管理。用「任务」来思考往往有用，但 Rust 本身没有「任务」这一概念，而且这个词在不同语境含义不同，容易混淆！更糟的是，运行时*有*自己的「任务」概念，不同运行时的定义还略有差别。

下文我会尽量精确使用与任务相关的术语。只说「任务」时，指可能与其它任务并发的一段抽象计算序列。「异步任务」含义相同，但与用 OS 线程实现的任务相对。「运行时的任务」指某运行时心目中的任务；「Tokio 任务」等则指具体运行时的任务。

Rust 中的异步任务就是 Future（通常是由许多小 Future 组合成的「大」Future）。换句话说，任务就是*被执行*的 Future。但有时 Future 被「执行」却并不是运行时的任务——直觉上它是*任务*，却不是*运行时的任务*。遇到例子时会再说明。

## 异步函数

`async` 关键字修饰函数声明，例如 `pub async fn send_to_server(...)`。**异步函数**就是用 `async` 声明的函数，含义是它可以被异步执行——换句话说，调用方*可以选择不*等函数完成就去做别的事。

更机械地说：调用异步函数时，函数体不会像普通函数那样立即执行，而是把函数体与参数打包成一个 Future，作为返回值代替真实结果。调用方再决定如何处理该 Future（若调用方想「立刻」要结果，就会 `await` 这个 Future，见下一节）。

在异步函数内部，代码仍按平常的*顺序*方式执行[^preempt]，是否 async 并无区别。可以在异步函数里调用同步函数，执行照常进行。异步函数里多出来的是可以用 `await` 等待其他异步函数（或 Future），这*可能*导致交出控制权，让其他任务执行。

[^preempt]: 与其他线程一样，运行异步函数的 OS 线程也可能被操作系统抢占、暂停以便其他线程工作。但从函数自身视角，若不检查可能被其他线程修改的数据，这种抢占是不可观察的（那些数据也可能在不停顿当前线程的情况下被并行修改）。

## `await`

前面说过，Future 是在将来某刻才会就绪的计算。要得到计算结果，使用 `await` 关键字。若结果已就绪或无需等待即可算出，`await` 就直接完成计算并给出结果；若尚未就绪，`await` 把控制权交给调度器，让其他任务继续（即上一章的协作式多任务）。

语法是 `some_future.await`，即作为后缀关键字与 `.` 运算符一起使用，因此在方法链和字段访问中可以很顺手地书写。

考虑以下函数：

```rust,norun
// An async function, but it doesn't need to wait for anything.
async fn add(a: u32, b: u32) -> u32 {
  a + b
}

async fn wait_to_add(a: u32, b: u32) -> u32 {
  sleep(1000).await;
  a + b
}
```

若调用 `add(15, 3).await`，会立刻得到 `18`。若调用 `wait_to_add(15, 3).await`，最终也是同样结果，但在等待期间其他任务有机会运行。

在这个简单例子里，`sleep` 代表需要等待结果的耗时操作，通常是 I/O：从外部读取数据，或确认写入外部目标成功。读取类似 `let data = read(...).await?`：此时 `await` 会让当前任务在读取进行期间等待，读取完成后任务恢复（等待期间其他任务可以工作）。读取结果可能是成功读到的数据，也可能是错误（由 `?` 处理）。

注意：若调用 `add`、`wait_to_add` 或 `read` 时不用 `.await`，不会得到任何答案！

什么？

调用异步函数返回的是 Future，不会立刻执行函数里的代码。而且 Future 在被 `await` 之前不会做任何事情[^poll]。这与某些语言不同——那些语言里 async 函数返回的 Future 会立即开始执行。

这是 Rust 异步编程的重要一点，熟悉后会成为本能，但常让新手踩坑，尤其是有其他语言异步经验的人。

Rust 中 Future 的一个重要直觉是：它们是*惰性*对象，必须由外部力量（通常是异步运行时）驱动才会真正干活。

我们把 `await` 描述得很「操作性」（运行 Future 得到结果），但上一章还谈了异步任务与并发——`await` 如何融入那套心智模型？先看纯顺序代码：逻辑上调用函数就是执行函数里的代码（并绑定变量），即当前任务继续执行由该函数定义的下一「块」代码。类似地，在异步上下文里调用非 async 函数，就是继续执行该函数；调用 async 函数则找到要运行的代码，但*不*运行它。`await` 是继续执行当前任务的运算符；若当前任务此刻无法继续，就让其他任务有机会继续。

`await` 只能在异步上下文中使用——目前指 async 函数内部（后面会看到更多异步上下文）。原因是 `await` 可能把控制权交给运行时，让其他任务执行，而只有异步上下文里才有运行时。暂时可以把运行时想成只在 async 函数里可访问的「全局变量」，后面会说明实际机制。

再从另一个角度看 `await`：前面提到 Future 可以组合成更大的 Future。`async` 函数是定义 Future 的一种方式，`await` 是组合 Future 的一种方式。在某个 async 函数里对 Future 使用 `await`，就把该 Future 组合进该 async 函数所产生的 Future 里。这种视角及其他组合方式后面会详述。

[^poll]: 或通过*轮询*（poll）——比 `await` 更底层的操作，使用 `await` 时在幕后发生。讨论 Future 细节时会再讲轮询。

## 一些 async/await 示例

先从「hello, world!」回顾：

```rust,edition2021
{{#include ../../examples/hello-world/src/main.rs}}
```

你现在应能认出 `main` 周围的样板：用于初始化 Tokio 运行时并创建初始任务来运行 async 的 `main`。

`say_hello` 是异步函数，调用后必须跟 `.await` 才能作为当前任务的一部分运行。注意：若去掉 `.await`，程序什么也不做！调用 `say_hello` 只返回 Future，从未执行，因此 `println` 不会被调用（编译器至少会警告）。

下面是一个稍贴近实际的例子，来自 [Tokio 教程](https://tokio.rs/tokio/tutorial/hello-tokio)。

```rust,norun
#[tokio::main]
async fn main() -> Result<()> {
    // Open a connection to the mini-redis address.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Set the key "hello" with value "world"
    client.set("hello", "world".into()).await?;

    // Get key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
```

代码更有趣一些，但本质相同——调用异步函数，再 `await` 以执行。这里用 `?` 做错误处理，与同步 Rust 一样。

尽管前面大谈并发、并行与异步，这两个例子都是 100% 顺序的。仅仅调用并 `await` 异步函数并不会引入并发，除非在等待期间还有其他任务可被调度。用另一个简单（人为的）例子自证：

```rust,edition2021
{{#include ../../examples/hello-world-sleep/src/main.rs}}
```

在打印 "hello" 与 "world" 之间，让当前任务睡眠[^async-sleep] 一秒。运行程序：先打印 "hello"，空等一秒，再打印 "world"。因为执行单个任务完全是顺序的。若有并发，这一秒的睡眠正是做别的事（比如先打印 "world"）的好机会。下一节会看到如何实现。

[^async-sleep]: 这里用的是异步 sleep。若用 std 的 [`sleep`](https://doc.rust-lang.org/std/thread/fn.sleep.html)，会把整个线程睡死。在这个玩具例子里差别不大，但在真实程序里意味着该线程上的其他任务在这段时间内无法被调度，非常糟糕。


## 生成任务

我们把 async/await 当作在异步任务里运行代码的方式，并说 `await` 可以在等待 I/O 或其他事件时让当前任务睡眠，此时其他任务可以运行——那些任务从哪来？就像用 `std::thread::spawn` 生成新线程任务，可以用 [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html) 生成新的异步任务。注意 `spawn` 属于 Tokio 运行时，而非标准库，因为「任务」纯粹是运行时概念。

下面是用 `spawn` 在单独任务上运行异步函数的小例子：

```rust,edition2021
{{#include ../../examples/hello-world-spawn/src/main.rs}}
```

与上一例类似，两个函数分别打印 "hello" 和 "world!"，但这次是并发（且并行）而非顺序执行。多运行几次，应能看到两种顺序——有时先 "hello"，有时先 "world!"，典型的并发竞态！

来看这里发生了什么。涉及三个概念：Future、任务与线程。`spawn` 接受一个 Future（记住可由许多小 Future 组成），把它作为新的 Tokio 任务运行。Tokio 调度和管理的是*任务*（不是单个 Future）。Tokio（默认配置下）是多线程运行时，因此生成新任务时，该任务可能在不同于生成方的 OS 线程上运行（也可能在同一线程，或先在一线程上跑再迁到另一线程）。

因此，Future 被 `spawn` 成任务后，会与生成它的任务及其他任务*并发*执行；若被调度到不同线程，还可能与那些任务*并行*。

概括：在 Rust 里写两条顺序语句，就是顺序执行（无论是否 async）。写 `await` 不会改变顺序语句的并发性。例如 `foo(); bar();` 严格顺序——先调用 `foo`，再调用 `bar`；无论 `foo`、`bar` 是否 async 皆然。`foo().await; bar().await;` 也严格顺序——`foo` 完全求值后再完全求值 `bar`。两种情况下，其他线程都可能与这段顺序执行交错；第二种情况下，在 `await` 点还可能与其他异步任务交错，但两条语句*彼此*仍是顺序的。

使用 `thread::spawn` 或 `tokio::spawn` 会引入并发，并可能引入并行——前者在线程之间，后者在任务之间。

本指南后面还会看到：有些场景下 Future 会并发执行，但永远不会并行执行。


### 合并（join）任务

若要拿到已生成任务的执行结果，生成方可以等待它结束并使用结果，这称为*合并*（join）任务（类比[合并](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html#method.join)线程，API 也类似）。

任务被 `spawn` 时，`spawn` 返回 [`JoinHandle`](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html)。若只想让任务自己跑，可以丢弃 `JoinHandle`（丢弃不影响已生成的任务）。若生成方要等被生成任务完成再用结果，可以对 `JoinHandle` 使用 `await`。

再回顾一次「Hello, world!」：

```rust,edition2021
{{#include ../../examples/hello-world-join/src/main.rs}}
```

代码与上次类似，但这次保存 `spawn` 返回的 `JoinHandle`，稍后再 `await`。因为我们在退出 `main` 前等待这些任务完成，`main` 里不再需要 `sleep`。

两个被生成的任务仍在并发执行，多运行几次仍应看到两种顺序。但被 `await` 的 `JoinHandle` 限制了并发：最后的感叹号（`!`）*总是*最后打印（可把 `println!("!");` 相对 `await` 的位置改改试试，可能还要改 sleep 时间才能观察到效果）。

若立刻 `await` 第一个 `spawn` 的 `JoinHandle`（写成 `spawn(say_hello()).await;`），则虽生成了运行 'hello' Future 的另一任务，但生成方会等它结束才做别的事——也就是说不可能有并发！几乎永远不该这样做（何必 spawn？直接写顺序代码即可）。

### `JoinHandle`

简要深入 `JoinHandle`。能对 `JoinHandle` 使用 `await`，说明 `JoinHandle` 本身也是 Future。`spawn` 不是 `async` 函数，而是返回 Future（`JoinHandle`）的普通函数；它在返回 Future 前会做一些工作（调度任务），因此*不必*对 `spawn` 本身 `await`。`await` `JoinHandle` 会等待被生成任务完成并返回结果。上例没有返回值，只等待完成。`JoinHandle` 是泛型，类型参数是被生成任务的返回类型；上例类型为 `JoinHandle<()>`，返回 `String` 的 Future 会得到 `JoinHandle<String>`。

`await` `JoinHandle` 得到的是 `Result`（上例用 `let _ = ...` 是为避免未使用 `Result` 的警告）。若被生成任务正常完成，结果在 `Ok` 中；若任务 panic 或被中止（一种[取消](../part-reference/cancellation.md)形式），结果为包含 [`JoinError`](https://docs.rs/tokio/latest/tokio/task/struct.JoinError.html) 的 `Err`。若项目不用 `abort` 做取消，对 `JoinHandle.await` 的结果 `unwrap` 是合理做法，相当于把被生成任务的 panic 传播到生成方。

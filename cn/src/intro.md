注意：本指南在长时间缺乏维护后正在进行重写。目前仍在编写中，许多内容尚未完成，已有部分也略显粗糙。

# 简介

本书是 Rust 异步编程指南，旨在帮助你迈出第一步，并进一步探索更高级的主题。我们不要求你具备异步编程经验（无论是在 Rust 还是其他语言中），但假定你已熟悉 Rust。如果你想学习 Rust，可以从[《Rust 程序设计语言》](https://doc.rust-lang.org/stable/book/)入手。

本书分为两大部分：[第一部分](part-guide/intro.md)是面向初学者的指南，建议按顺序阅读，带你从零基础提升到中级水平。第二部分则是若干独立章节，涵盖更高级的主题；在完成第一部分之后，或若你已有一定的 async Rust 经验，第二部分会很有参考价值。

你可以通过多种方式浏览本书：

* 从头到尾按顺序阅读。对于 async Rust 新手，至少建议对本书的[第一部分](part-guide/intro.md)采用这种方式。
* 网页左侧有目录摘要。
* 若想了解某个宽泛主题，可从主题索引入手。
* 若想查找关于某一具体话题的全部讨论，可从详细索引入手。
* 也可查看常见问题（FAQ）中是否已有解答。


## 什么是异步编程，以及为何使用它？

在并发编程中，程序会同时（或至少看起来同时）做多件事。使用线程编程是并发编程的一种形式：线程内的代码以顺序风格编写，由操作系统并发执行各线程。而在异步编程中，并发完全发生在你的程序内部（操作系统不参与）。异步运行时（在 Rust 中只是另一个 crate）负责管理异步任务，程序员则通过 `await` 关键字显式让出控制权。

由于不涉及操作系统，异步世界中的*上下文切换*非常快。此外，异步任务的内存开销远低于操作系统线程。因此，异步编程很适合需要处理大量并发任务、且这些任务大量时间在等待（例如等待客户端响应或 I/O）的系统；也适合内存极为有限、且没有提供线程的操作系统的微控制器。

异步编程还让程序员能细粒度控制任务的执行方式（并行与并发程度、控制流、调度等）。这意味着对许多场景而言，异步编程既富有表现力又便于使用。尤其是 Rust 中的异步编程具有强大的取消（cancellation）概念，并支持多种并发风格（通过 `spawn` 及其变体、`join`、`select`、`for_each_concurrent` 等构造表达），从而可以组合、复用超时、暂停、限流等模式的实现。


## Hello, world!

为让你对 async Rust 有个直观感受，下面是一个「hello, world」示例。其中没有并发，也并未真正发挥异步的优势；但它定义并调用了异步函数，并会打印 "hello, world!"：

```rust,edition2021
{{#include ../examples/hello-world/src/main.rs}}
```

后续章节会详细解释一切。目前请注意：我们用 `async fn` 定义异步函数，用 `.await` 调用——在 Rust 中，异步函数若未被 `await`，则不会执行任何逻辑[^blocking]。

与本书所有示例一样，若你想查看完整示例（例如包含 `Cargo.toml`）或在本地自行运行，可在本书的 GitHub 仓库中找到，例如 [examples/hello-world](https://github.com/rust-lang/async-book/tree/master/examples/hello-world)。


## Async Rust 的发展现状

Rust 的异步特性已开发多年，但语言中这一部分尚未「完工」。Async Rust（至少稳定编译器与标准库中可用的部分）可靠且性能良好，已在一些大型科技公司的最严苛生产场景中投入使用。不过仍有一些缺失与粗糙之处（这里的「粗糙」主要指易用性，而非可靠性）。在 async Rust 的学习与实践过程中，你很可能会遇到其中一些；对大多数缺失部分，本书会介绍相应的变通做法。

目前，多数用户感到不够顺手的是异步迭代器（亦称 stream）。trait 中的部分 async 用法尚未得到良好支持，异步析构（destruction）也尚无理想方案。

Async Rust 仍在积极开发中。若想关注进展，可访问 Async 工作组[主页](https://rust-lang.github.io/wg-async/meetings.html)，其中包含其[路线图](https://rust-lang.github.io/wg-async/vision/roadmap.html)；也可阅读 Rust 项目中的 async [项目目标](https://github.com/rust-lang/rust-project-goals/issues/105)。

Rust 是开源项目。若你希望为 async Rust 的开发做贡献，可从主 Rust 仓库的[贡献指南](https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md)开始。


[^blocking]: 这其实是个不太好的示例，因为 `println` 属于*阻塞 I/O*，在异步函数中做阻塞 I/O 通常并不合适。我们将在 [chapter TODO]() 中说明什么是阻塞 I/O，并在 [chapter TODO]() 中解释为何不应在异步函数中进行阻塞 I/O。

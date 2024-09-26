# 22-并行性：第一部分 - 你想构建一个语言虚拟机吗？

- [22-并行性：第一部分 - 你想构建一个语言虚拟机吗？](#22-并行性第一部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [问题](#问题)
        - [GILs 和 OS 线程](#gils-和-os-线程)
        - [BEAM](#beam)
            - [参与者 (Actors)](#参与者-actors)
        - [Iridium](#iridium)
            - [第 1 步：核心感知 (Core Awareness)](#第-1-步核心感知-core-awareness)
            - [第 2 步：添加 CLI 标志](#第-2-步添加-cli-标志)
        - [专有名词及注释](#专有名词及注释)

## 引言

您好！在这篇文章中，我们将重点介绍开始向 Iridium 虚拟机添加并行性 (`parallelism`)。

| |  |
| --- | --- |
| 重要提示 | 如果您不确定并行性 (`parallelism`) 和并发性 (`concurrency`) 之间的区别，请先阅读这篇文章：https://blog.subnetzero.io/article/concurrency-vs-parallelism/|

## 问题

Iridium 虚拟机使用单个操作系统线程。这与 CPython、标准 Ruby 和其他类似语言虚拟机的运作方式相似。它们使用一种称为全局解释器锁（GIL）的东西；理解它们的工作原理很重要，所以我们从这里开始。

### GILs 和 OS 线程

如果我们像这样在 Python 中创建一个线程：

```python
from thread import start_new_thread

def some_func():
    pass

start_new_thread(some_func,())
c = raw_input("Type something to quit.")
```

我们将得到一个真正的操作系统线程。逻辑上假设如果我们创建了更多的线程，我们就可以完成更多的工作，对吧？但恐怕这不是真的。=(

即使我们有多个线程，仍然必须有 _某种东西_ 执行字节码。在 Python 中，那就是 Python 虚拟机。GIL 是一个互斥锁（或者是锁），它 _只允许一个线程一次使用 Python 虚拟机_。这是出于一个很好的原因：它有助于防止底层 Python 虚拟机中的竞态条件。所以即使你有 10 个操作系统线程，你仍然只能使用一个核心。

### BEAM

BEAM 虚拟机（或 Erlang 虚拟机），这是 Iridium 的主要灵感 (`inspiration`) 来源，以一种允许并行性的不同方式解决了数据竞争问题。那个范式称为 _参与者_(_`Actor`_) 模型。

当在 Erlang 或 Elixir 中编写代码时，你将创建 _进程_。这些 _不是_ 操作系统进程，而是 BEAM 进程。其他术语还包括绿色线程 (`green threads`)、微线程 (`micro threads`)、轻量级线程 (`lightweight threads`)、协程 (`co-routines`) 和纤程 (`fibers`)。所有这些术语都有不同的细微含义，但大致指的是同一件事，即由语言虚拟机管理的非操作系统线程。

#### 参与者 (Actors)

在 BEAM 中，每个 _进程_ 都与其他所有进程隔离。将它们想象为在电子景观 (`electronic landscape`) 中独立行走的实体 (`entities`)。这里是重要的部分：参与者只能通过消息传递 (`message passing`) 彼此通信。


> If we have process A and B, A cannot reach into B and do something like B.count = 10. It has to send a message to B telling B to change its count to 10. The result of this is that the only thing that can modify an Actor’s internal state is that Actor, thus preventing race conditions. Another effect of this is that an Actor can run on any core and be moved about without issues, even across networks.

如果我们有进程 A 和 B，A 不能进入 B 并做类似 `B.count = 10` 的事情。它必须向 B 发送一条 _消息_，告诉 B 更改其计数为 10。这个结果就是唯一可以修改参与者内部状态的是参与者本身，从而防止了竞态条件。另一个效果是，参与者可以在任何核心上运行，并可以在没有问题的情况下移动，甚至跨网络。

缺点 (`downside`) 是它比直接共享内存 (`sharing memory directly`) 慢。

### Iridium

在 Iridium 中，我们将遵循 BEAM 模型。这是一个大项目，所以我们不会在本教程中全部完成。我们将开始奠定基础 (`We’ll start laying the foundation.`)。

#### 第 1 步：核心感知 (Core Awareness)

让我们先让 Iridium 感知到它运行的机器拥有的核心数量。有一个方便 (`handy-dandy`) 的 crate 可以做到这一点：<https://github.com/seanmonstar/num_cpus>。去吧，将 `num_cpus = "1.0"` 添加到 Cargo.toml 并将 `extern crate num_cpus;` 添加到 `bin/iridium.rs` 和 `lib.rs`。

现在，我们将在 VM 中添加一个属性，包含它检测到的核心数量。在 `src/vm.rs` 中添加这个属性：

```rust
pub logical_cores: usize,
```

并在 VM impl 的 `new` 函数中，在创建 VM 结构体时添加：

```rust
logical_cores: num_cpus::get(),
```

#### 第 2 步：添加 CLI 标志

既然我们在这里，让我们给用户提供一种方法来设置 VM 可以使用的线程数量。前往 `src/bin/cli.yml` 并添加这个参数：

```yaml
- THREADS:
    help: Number of OS threads the VM will utilize
    required: false
    takes_value: true
    long: threads
    short: t
```

在 `bin/iridium.rs` 中，在检查 `target_file` 之前，添加这个：

```rust
let num_threads = match matches.value_of("THREADS") {
    Some(number) => {
        match number.parse::<usize>() {
            Ok(v) => { v }
            Err(_e) => {
                println!("Invalid argument for number of threads: {}. Using default.", number);
                num_cpus::get()
            }
        }
    }
    None => {
        num_cpus::get()
    }
};

```

我们需要做的最后一件事是在运行文件（而不是 REPL）时更改核心计数：

```rust
match target_file {
    Some(filename) => {
        let program = read_file(filename);
        let mut asm = Assembler::new();
        let mut vm = VM::new();
        vm.logical_cores = num_threads;
        // <snip>
```

现在如果我们编译并运行 `iridium --help`，我们会看到：

```sh
Interpreter for the Iridium language

USAGE:
    iridium [OPTIONS] [INPUT_FILE]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -t, --threads <THREADS>    Number of OS threads the VM will utilize

ARGS:
    <INPUT_FILE>    Path to the .iasm or .ir file to run
```

耶！

原文出处：[Parallelism: Part 1 - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-22/)
作者名：Fletcher

### 专有名词及注释

1. 全局解释器锁 (GIL): Python 虚拟机中的锁，它只允许一个线程使用虚拟机。它是为了防止底层虚拟机中的竞态条件而存在的。
描述中的术语和定义存在一些混淆和不准确的地方。下面是对这些术语的更准确的解释和完善：
2. 绿色线程 (`Green Threads`)
    - **定义**：绿色线程是由用户空间库管理的线程，而不是由操作系统内核调度。这些线程通常比传统的内核线程更轻量级，因为它们不需要内核的上下文切换开销。
    - **特点**：
        - 绿色线程可以在单个操作系统线程中实现多任务并发。
        - 调度完全由应用程序或库控制，而不是操作系统内核。
        - 由于没有内核参与调度，因此它们具有更高的性能和更低的上下文切换开销。
    - **例子**：Java 虚拟机（JVM）早期版本中的线程调度就是一个典型的绿色线程实现。
3. 微线程 (`Microthreads` or `Fiber` 可能更为准确)
    - **定义**：微线程指的是在用户空间中创建和管理的轻量级线程，类似于绿    色线程，但是更加轻量级。
    - **特点**：
      - 微线程通常用于高并发场景，例如 Web 服务器、网络应用等。
      - 它们可以由库或框架自动管理和调度，无需内核干预。
      - 每个微线程可以独立执行，但通常共享相同的内存空间。
    - **例子**：Go 语言中的 goroutine 可以被视为一种微线程的实现。
4. 轻量级线程 (`Lightweight Threads`)
    - **定义**：轻量级线程通常指的是一种在用户空间或内核空间中实现的相对    较小的线程模型。
    - **特点**：
      - 相比于传统的线程，轻量级线程的上下文切换成本更低。
      - 它们可以在一个单一的系统线程中运行多个轻量级线程。
      - 轻量级线程可以由内核管理，也可以由用户空间的库管理。
    - **例子**：Linux 中的 LWP（轻量级进程）或 Windows 中的 fibers。
5. 协程 (`Co-routines`)
    - **定义**：协程是一种可以在多个执行点之间相互挂起和恢复的程序结构。
    - **特点**：
        - 协程可以在不同的执行点之间进行控制权的转移，而不仅仅是简单的函数调用。
        - 协程可以在多个执行路径之间切换，但它们通常共享相同的内存空间。
        - 协程可以由用户空间的库或语言特性实现，也可以由操作系统支持。
    - **例子**：Python 中的 `asyncio` 模块使用协程来实现异步编程。
6. 纤程 (`Fibers`)
    - **定义**：纤程是一种用户空间或内核空间的轻量级线程模型，允许程序在多个执行点之间切换。
    - **特点**：
      - 纤程可以由用户空间的库或内核实现。
      - 它们可以手动或自动调度，并且可以保存和恢复执行状态。
      - 纤程可以拥有自己的栈空间，并且可以在不同执行点之间暂停和恢复。
    - **例子**：Windows 中的 `CreateFiber` 和 `SwitchToFiber` 函数可以用来创建和切换纤程。
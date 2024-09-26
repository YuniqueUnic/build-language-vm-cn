# 17-基础线程 - 你想构建一个语言虚拟机吗？

- [17-基础线程 - 你想构建一个语言虚拟机吗？](#17-基础线程---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [关于假设知识的说明](#关于假设知识的说明)
    - [多线程 (Multithreading)](#多线程-multithreading)
    - [线程](#线程)
        - [REPL 流程](#repl-流程)
        - [可审核性和 PIDs (Auditability and PIDs)](#可审核性和-pids-auditability-and-pids)
    - [稍微离题](#稍微离题)
        - [一次测试](#一次测试)
    - [回到线程](#回到线程)
        - [函数签名](#函数签名)
        - [还是线程](#还是线程)
        - [将调度器添加到 REPL](#将调度器添加到-repl)
            - [汇编器](#汇编器)
        - [第一个新函数](#第一个新函数)
        - [测试](#测试)
        - [一个意外的失败](#一个意外的失败)
    - [总结](#总结)
    - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

大家好！在本教程中，我们将开始为 Iridium 虚拟机添加多线程功能。请确保你从这个代码点开始：[0.0.16 标签](https://gitlab.com/subnetzero/iridium/tags/0.0.16)。继续前进，我将为每个教程制作一个标签，以便每个人都从一个共同的起点开始。

### 关于假设知识的说明

我写的这些教程针对的是更高级的用户。我有时会跳过一些小步骤，比如“在文件 X 中添加这一行”。

## 多线程 (Multithreading)

当前版本的 Iridium 是单进程、单线程的。当你执行一个应用程序时，虚拟机在完成之前不能做其他任何事情。这是像 Python 和 Ruby 这样的语言的 VMs 的工作方式。我们希望 Iridium 的工作方式更像 BEAM VM，它为 VM 提供了一种外壳。它可以查看正在运行的进程、终止它们或其他管理任务。

这当然需要不止一个教程部分，当然。但是一个好的第一步应该是能够使用 REPL 在单独的 OS 线程中运行一个程序。

准备好了吗？我们开始吧！

## 线程

我们可能首先需要弄清楚如何在 Rust 中创建线程。Rust 书籍很好地涵盖了它们，所以我将从中偷一个示例：

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

我们能否添加一个 REPL 命令，接受汇编代码文件的路径，编译它，然后将其交给后台线程中的 VM？会不会就这么简单呢？！让我们来找出答案！

### REPL 流程

在继续之前，我们可能需要先考虑一下这个细节。这是一条稍微更《无聊》的路径，但我们的用户会感谢我们的。

_也许吧。_

好的，可能不会。

无论如何，这里有一个可能的工作流程：

```sh
欢迎使用 Iridium！让我们提高效率！
>>> .spawn
请输入你希望加载的文件路径：test.iasm
在后台线程中启动程序
>>>
```

思考这个问题时，出现了一些问题：

1. 我们如何获取程序的输出？

2. 我们如何将程序的输出显示给用户？

3. 我们需要跟踪在后台运行的所有程序吗？

4. 我们将如何允许这些程序获取输入？

我相信我们还能想到更多，但让我们先解决跟踪程序的问题。

### 可审核性和 PIDs (Auditability and PIDs)

每当你在运行 Linux 的计算机上执行程序时，操作系统都会给它一个叫做 `PID` 或 _进程标识符_ 的东西。Linux 保证 PID 在当前运行的进程中是唯一的，非负的，并且在 1 到 32,767 之间。当它达到这个数字后，它将开始循环。如果一个进程启动，得到一个 PID 为 600，运行，然后停止，那么 600 可以被重新使用。

| | |
|---|---|
| 注意 | 为什么是那么具体的上限？因为那是大多数 Linux 内核的默认值。随着计算需求和能力的发展，一个服务器运行超过 32k 个进程的想法不再像以前那样荒谬。为了适应这一点，你可以将最大 PID 改为大约四百万。|

当我们启动一个 REPL 会话时，它被视为 **一个进程**。我们可以在后台启动 OS 线程，但操作系统并不知道 Iridium VM 正在做什么的具体信息。如果我们想要跟踪 VM 运行的代码和结果，我们将不得不做同样的事情。

## 稍微离题

当我在 0.0.16 版本中四处探索时，发生了这种情况：

```sh
>>> .load_file
请输入你希望加载的文件路径：
尝试从文件加载程序...
thread 'main' panicked at 'File not found: Os { code: 2, kind: NotFound, message: "No such file or directory" }', libcore/result.rs:945:5
```

看来当 REPL 找不到文件时会崩溃。我们将迅速修复这个问题。

| | |
|---|---|
| 注意 | 由于我们正在进入对我来说的新领域，我们可能会有很多这样的小侧支任务。=)|

有问题的代码行是 `src/repl/mod.rs` 中的这一行：

```rust
let mut f = File::open(Path::new(&filename)).expect("File not found");
```

让我们改变它，以在不崩溃的情况下处理错误情况：

```rust
let filename = Path::new(&tmp);
let mut f = match File::open(&filename) {
    Ok(f) => { f }
    Err(e) => {
        println!("There was an error opening that file: {:?}", e);
        continue;
    }
};
```

我们还可以消除那个双重 `Path::new()` 调用。

### 一次测试

在做出这个改变后，所有测试仍然通过，如果我们现在尝试给它一个不存在或错误的文件名，我们会得到：

```sh
欢迎使用 Iridium！让我们提高效率！
>>> .load_file
请输入你希望加载的文件路径：
尝试从文件加载程序...
无法打开文件：Os { code: 2, kind: NotFound, message: "没有这样的文件或目录" }
>>> .load_file
请输入你希望加载的文件路径：doh
尝试从文件加载程序...
无法打开文件：Os { code: 2, kind: NotFound, message: "没有这样的文件或目录" }
>>>
```

太好了！提交这个，我们可以回到线程。

## 回到线程

让我们先处理一下生成线程的事情。创建一个新模块，`src/scheduler/mod.rs`。我们使用一个新模块，因为我怀疑这将是一个我们以后会使用的更复杂调度的开始。我们还可以使用 `Scheduler` 结构来跟踪信息，例如 PID。

在新模块中，放入以下内容：

```rust
use std::thread;
use vm::VM;

#[derive(Default)]
pub struct Scheduler {

}

impl Scheduler {
    pub fn new() -> Scheduler {
        Scheduler{}
    }

    pub fn get_thread(vm: VM) {

    }
}
```

### 函数签名

让我们看一下 thread::spawn() 函数的签名：

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
```

我会逐行讲解；这里引入了一些新的高级 Rust 概念。

首先，注意到它对 `F` 和 `T` 是泛型的，并且返回一个 `JoinHandle<T>`。什么是 `JoinHandle<T>` 你问？好问题！你可以将它们视为正在执行的线程的句柄。它 **不是** 线程本身，也不是它的简单指针。无论线程做什么，它都必须返回 `T`。

类型参数 `F` 是一个 FnClose，或函数闭包 (`function closure`)。这里的 `where` 约束表明允许哪些类型的函数：

```rust
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
```

它们必须实现 `Send + ’static'`，和 `FnOnce() → T`。这意味着两件事：

1. 函数必须返回 T

2. FnOnce 函数被调用一次（有其他类型在需要时，例如 Fn, FnMut）

3. 要求 'static 意味着它将存在于程序的生命周期内。注意：这意味着 _闭包_，而不是它的特定执行。

这里有一个简单的例子：

```rust
let join_handle: thread::JoinHandle<u32> = thread::spawn(|| {
    10.0    // 这会失败，因为它是一个浮点数，而不是一个 u32
    10      // 这会成功，因为它是一个 u32，而不是一个浮点数
});
```

| | |
|---|---|
| 注意 | 在文档中，你可能会看到 `JoinHandler<T>` 被写为 `JoinHandler<_>`。`_` 是一个占位符，不会编译（我认为，无论如何）。|

这意味着我们必须改变我们的 `VM::run()` 函数以返回一个值。简单至极 (`Easy-peasy`)：

```rust
/// 将执行包装在一个循环中，这样它将继续运行，直到完成或执行指令时出错。
/// 执行指令集
pub fn run(&mut self) -> u32 {
    // TODO: 应该在这里设置自定义错误
    if !self.verify_header() {
        println!("Header was incorrect");
        return 1;
    }

    // 如果头部有效，我们需要将 PC 更改为位 65。
    self.pc = 65;
    let mut is_done = false;
    while !is_done {
        is_done = self.execute_instruction();
    }
    0
}
```

### 还是线程

回到生成池，zergling！

实际的线程代码本身很简单：`vm.run()`。`get_thread` 将接受一个 VM，创建一个线程，该线程将执行 vm.run() 直到它返回一个值。这很简单，因为我们给线程提供了一个完整的 VM
 的所有权，所以 `borrowck` 仍然满意。随着我们使调度程序更加高级，我们将不得不变得更有创意。目前，这将有效：

```rust
impl Scheduler {
    pub fn new() -> Scheduler {
        Scheduler{}
    }

    /// 接受一个 VM 并在后台线程中运行它
    pub fn get_thread(mut vm: VM) -> thread::JoinHandle<u32> {
      thread::spawn(move || {
          vm.run()
      })
    }
}
```

现在，让我们复制 Linux 模型，并从 0 开始为每个程序分配一个唯一的 PID。让我们像这样改变我们的 Scheduler 结构：

```rust
pub struct Scheduler {
    next_pid: u32,
    max_pid: u32,
}

impl Scheduler {
    pub fn new() -> Scheduler {
        Scheduler{
            next_pid: 0,
            max_pid: 50000
        }
    }
}
```

### 将调度器添加到 REPL

就像我们给 REPL shell 它自己的 VM 一样，我们可以给它一个调度器，像这样：

```rust
use scheduler::Scheduler;

/// 汇编器的 REPL 的核心结构
pub struct REPL {
    command_buffer: Vec<String>,
    vm: VM,
    asm: Assembler,
    scheduler: Scheduler
}

impl REPL {
    /// 创建并返回一个新的汇编 REPL
    pub fn new() -> REPL {
        REPL {
            vm: VM::new(),
            command_buffer: vec![],
            asm: Assembler::new(),
            scheduler: Scheduler::new()
        }
    }

```

我不会再次包括 REPL 函数的其余部分。你可以向上滚动查看它们。=)

#### 汇编器

目前，我们仍然让 REPL 处理汇编，并给线程一个准备运行的字节码的 VM。

| | |
|---|---|
| 重要 | ".spawn" 和 ".load_file" 命令几乎相同。让我们将它们分解成更小的函数。|

### 第一个新函数

```rust
fn get_data_from_load(&mut self) -> Option<String> {
    let stdin = io::stdin();
    print!("请输入你希望加载的文件路径：");
    io::stdout().flush().expect("无法刷新 stdout");
    let mut tmp = String::new();

    stdin.read_line(&mut tmp).expect("无法从用户读取行");
    println!("尝试从文件加载程序...");

    let tmp = tmp.trim();
    let filename = Path::new(&tmp);
    let mut f = match File::open(&filename) {
        Ok(f) => { f }
        Err(e) => {
            println!("打开文件时出错：{:?}", e);
            return None;
        }
    };
    let mut contents = String::new();
    match f.read_to_string(&mut contents) {
        Ok(program_string) => {
            Some(program_string.to_string())
        },
        Err(e) => {
            println!("读取文件时出错：{:?}", e);
            None
        }
    }
}
```

将其放入 `repl/mod.rs` 作为 REPL impl 的一部分。现在我们可以将 .spawn 和 .load_file 做得更小：

```sh
".spawn" => {
    let contents = self.get_data_from_load();
    if let Some(contents) = contents {
        match self.asm.assemble(&contents) {
            Ok(mut assembled_program) => {
                println!("Sending assembled program to VM");
                self.vm.program.append(&mut assembled_program);
                println!("{:#?}", self.vm.program);
                self.scheduler.get_thread(self.vm.clone());
            },
            Err(errors) => {
                for error in errors {
                    println!("Unable to parse input: {}", error);
                }
                continue;
            }
        }
    } else { continue; }
}
```

### 测试

我们可以从一个只有一个指令 `HLT` 的程序开始。你可以在 `docs/examples/iasm` 下找到它。让我们看看会发生什么！

```sh
>>> .spawn
Please enter the path to the file you wish to load: /Users/fletcher/Projects/iridium-book/docs/examples/iasm/hlt.iasm
Attempting to load program from file...
There was an error parsing the code: Error(Code(CompleteStr("4"), Many1))
Unable to parse input: There was an error parsing the code: Error(Code(CompleteStr("4"), Many1))
```

哎呀，是时候调试了。

<时间流逝> (`<time passed>`)

啊哈！你会在这一部分注意到：

```rust
match f.read_to_string(&mut contents) {
    Ok(program_string) => {
        Some(program_string)
    },
    Err(e) => {
        println!("there was an error reading that file: {:?}", e);
        None
    }
}
```

`read_to_string` 返回它读取的字节数，并将内容读入作为参数提供的 `String` 中。在我们的 `match` 语句中，我们将读取的字节数转换成了一个字符串并返回了。这就是解析器拒绝解析的内容。

修复方法很简单：

```rust
match f.read_to_string(&mut contents) {
    Ok(_bytes_read) => {
        Some(contents)
    },
    Err(e) => {
        println!("there was an error reading that file: {:?}", e);
        None
    }
}
```

让我们再一次尝试：

```sh
Welcome to Iridium! Let's be productive!
>>> .spawn
Please enter the path to the file you wish to load: docs/examples/iasm/hlt.iasm
Attempting to load program from file...
Loaded conents: Some(
    ".data\n\n.code\nload $0 #100\nhlt\n"
)
Did not find any errors in the first phase
Sending assembled program to VM
[
    45,
    50,
    49,
    45,
    0,
    0,
  <snip a ton of zeros>
    0,
    0,
    100,
    5,
    0,
    0,
    0
]
>>> thread '<unnamed>' panicked at 'index out of bounds: the len is 72 but the index is 72', /Users/travis/build/rust-lang/rust/src/libcore/slice/mod.rs:2079:10
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

经过我们的汇编器添加了头部等之后，最终程序的大小至少为 65 字节 (`bytes`)。64 个字节用于头部，至少 1 个字节用于指令。这看起来像是我们的程序计数器的[差一 (`off-by-one`) 错误](https://en.wikipedia.org/wiki/Off-by-one_error)。也就是说，程序向量的总长度为 72，执行循环中偏移了一个。我怀疑……

检查 `vm.rs` 中的 `run` 函数。

```rust
/// 将执行包装在一个循环中，这样它将继续运行，直到完成或执行指令时出错。
pub fn run(&mut self) -> u32 {
    // TODO: 应该在这里设置自定义错误
    if !self.verify_header() {
        println!("Header was incorrect");
        return 1;
    }
    // 如果头部有效，我们需要将 PC 更改为位 65。
    self.pc = 64;
    let mut is_done = false;
    while !is_done {
        is_done = self.execute_instruction();
    }
    0
}
```

看到它将程序计数器预设为 65 了吗？这意味着当 VM 开始执行时，它做的第一件事就是请求 _下一个_ 指令，即 66。VM 总是超前一步。

将其更改为 64，让我们再次尝试……

```sh
>>> .spawn
请输入你希望加载的文件路径：docs/examples/iasm/hlt.iasm
尝试从文件加载程序...
读取内容：Some(
    ".data\n\n.code\nload $0 #100\nhlt\n"
)
第一阶段没有发现任何错误
将组装好的程序发送到 VM
[
    45,
    50,
    49,
    45,
    0,
    0,
  <snip many zeros>
    0,
    0,
    100,
    5,
    0,
    0,
    0
]
>>> HLT encountered
```

嘿，它运行起来了，在后台线程中！让我们检查我们的寄存器，看看我们是否能看到值：

```sh
.registers
列出寄存器及其所有内容：
[
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0
]
寄存器列表结束
```

**什么鬼？**

因为 VM 在后台线程中运行，它有自己的一组寄存器和其他所有东西。当我们在 REPL 中键入 `.registers` 时，我们仍然在查看由主线程中的 REPL 创建和使用的 VM 的寄存器。

在后台线程中，当 `run` 终止时，所有这些数据结构都消失了……像雨中的眼泪一样。

（对不起但不抱歉）

### 一个意外的失败

想象一下，当我在这一点上运行 `cargo test` 时，它显示了四个失败的测试：test_sub_opcode, test_mul_opcode, test_div_opcode, test_add_opcode。错误是：

```sh
failures:

---- vm::tests::test_add_opcode stdout ----
thread 'vm::tests::test_add_opcode' panicked at 'index out of bounds: the len is 69 but the index is 69', /Users/travis/build/rust-lang/rust/src/libcore/slice/mod.rs:2079:10
note: Run with `RUST_BACKTRACE=1` for a backtrace.

---- vm::tests::test_div_opcode stdout ----
thread 'vm::tests::test_div_opcode' panicked at 'index out of bounds: the len is 69 but the index is 69', /Users/travis/build/rust-lang/rust/src/libcore/slice/mod.rs:2079:10

---- vm::tests::test_load_opcode stdout ----
Illegal instruction encountered
thread 'vm::tests::test_load_opcode' panicked at 'assertion failed: `(left == right)`
  left: `1`,
 right: `500`', src/vm.rs:336:9

---- vm::tests::test_mul_opcode stdout ----
thread 'vm::tests::test_mul_opcode' panicked at 'index out of bounds: the len is 69 but the index is 69', /Users/travis/build/rust-lang/rust/src/libcore/slice/mod.rs:2079:10

---- vm::tests::test_sub_opcode stdout ----
thread 'vm::tests::test_sub_opcode' panicked at 'index out of bounds: the len is 69 but the index is 69', /Users/travis/build/rust-lang/rust/src/libcore/slice/mod.rs:2079:10
```

再次见到你，差一错误 (`off by one error`)，我的老朋友。_我们再次见面！_

为了修复它，我不得不对 `vm.rs` 测试模块中的 `prepend_header` 进行小幅调整：

```rust
fn prepend_header(mut b: Vec<u8>) -> Vec<u8> {
    let mut prepension = vec![];
    for byte in PIE_HEADER_PREFIX.into_iter() {
        prepension.push(byte.clone());
    }
    while prepension.len() < PIE_HEADER_LENGTH {
        prepension.push(0);
    }
    prepension.append(&mut b);
    prepension
}

```

你能发现区别吗？=)

## 总结

我们发现并修复了一些错误，并成功地以一种我们可以在以后的基础上构建的原始形式在后台线程中运行应用程序。在下一部分，我们将完成 PID 跟踪。

这才只是我们将构建到我们的 VM 中的酷功能的开始。=) 你可以在 [GitLab 的 0.0.17 标签](https://gitlab.com/subnetzero/iridium/-/tree/0.0.17?ref_type=tags)下找到本教程后的代码的最终形式。

## 原文链接及作者信息

原文链接：[Basic Threads - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-17/)

作者名称：Fletcher

### 成语及典故

1. `A Faustian Bargain`：`浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。
2. `Zergling`: 是来自于游戏《星际争霸》（StarCraft）中的一个单位，属于 Zerg 种族的基本战斗单位。在这里，作者可能是以幽默的方式，将线程比作是快速、大量生产的小型战斗单位，以此来形象地描述线程的创建和管理工作。在翻译时，可以保留原文`zergling`，因为它已经成为了游戏文化中的一个专有名词，或者可以解释性地翻译为`快速生成的战斗单位`或`轻量级线程`。

### 专有名词及注释

1. 多线程 (Multithreading)：多线程是指一个程序可以同时执行多个任务，每个任务都运行在一个单独的线程中。
2. 线程 (Thread)：线程是操作系统中用于执行程序代码的基本执行单位，每个线程都有自己的内存空间和栈，可以独立地执行任务。
3. 线程池 (Thread Pool)：线程池是多个线程的集合，用于执行一些任务，每个任务由一个线程来完成。
4. 线程安全 (Thread Safety)：线程安全是指在多线程环境下，程序能够正确地运行，并且不会出现数据竞争、死锁等错误。
5. 互斥锁 (Mutex)：互斥锁是一种用于保护共享资源的并发访问机制，确保同一时间只有一个线程可以访问共享资源。
6. 死锁 (Deadlock)：死锁是指两个或多个线程互相等待对方释放资源，导致无法继续运行的情况。
7. 调度器 (Scheduler)：调度器是操作系统用于管理线程的组件，负责确定哪个线程应该被执行，以及何时执行。
8. [差一错误 (Off-by-one Error)](https://en.wikipedia.org/wiki/Off-by-one_error): 差一错误指的是程序在处理数据时，由于索引或指针的计算错误导致访问了数据 beyond the end of the array 或 beyond the end of the string 等。

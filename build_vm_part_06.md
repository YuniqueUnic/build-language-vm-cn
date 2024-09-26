# 06-REPL - 你想构建一个语言虚拟机吗？

- [06-REPL - 你想构建一个语言虚拟机吗？](#06-repl---你想构建一个语言虚拟机吗)
    - [基本结构](#基本结构)
        - [所有权 (Ownership)](#所有权-ownership)
    - [实现](#实现)
        - [REPL 结构](#repl-结构)
        - [命令缓冲区 (Command Buffer)](#命令缓冲区-command-buffer)
        - [虚拟机](#虚拟机)
        - [另一个循环……](#另一个循环)
            - [刷新 stdout](#刷新-stdout)
            - [main.rs](#mainrs)
        - [命令历史](#命令历史)
            - [历史命令](#历史命令)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

`REPL` 代表 **读取、评估和打印循环**。它也被称为语言的 **交互式解释器**。是交互式解释器，用于执行代码，它通常以交互的方式工作，允许用户输入和输出。

它是一种简单的、交互式的编程环境，可以让用户输入表达式（代码），然后系统立即计算并返回结果。REPL 环境在编程语言的学习、调试和原型设计中非常有用。

例如，如果你打开 Terminal 或 iTerm，我们可以看看 Python 的 REPL：

```sh
fletcher$ python
Python 2.7.10 (default, Oct 6 2017, 22:29:07)
[GCC 4.2.1 Compatible Apple LLVM 9.0.0 (clang-900.0.31)] on darwin
输入 "help", "copyright", "credits" 或 "license" 以获取更多信息。
>>> x = 1
>>> 1 + x
2
>>>
```

它允许你执行 Python 代码并查看结果，无需将代码写入 `.py` 文件。如果你想做一些快速的脚本编写或测试，这可能会很有用。我们的 REPL 将允许我们执行代码并与虚拟机交互，随着我们扩展其功能。

因为我们还没有汇编器或编译器，我们将不得不以十六进制输入我们的代码……但嘿，你不是想学习底层的东西吗？=)

## 基本结构

所有 `REPL` 都是一个无限循环，执行类似以下操作：

1. 等待输入 (`Wait for input`)
2. 评估输入 (`Evaluate input`)
3. 打印响应 (`Print response`)

那么，我们应该如何在我们的代码中构建这个结构呢？最常见的方法是将其作为语言解释器的一部分。这就是为什么如果你在没有文件名的情况下输入 `python` 或 `perl` 或 `ruby`，你会得到一个 REPL。

### 所有权 (Ownership)

虚拟机拥有 REPL 吗？REPL 拥有虚拟机吗？我们应该创建另一个数据结构来管理两者之间的交互吗？都是好问题！由于 Rust 的所有权模型，我们必须更多地考虑哪个数据结构拥有哪个其他数据结构。我采取的方法，到目前为止似乎运作良好，是让 REPL 管理一个虚拟机。

## 实现

首先，让我们在 `src/` 目录中创建一个名为 `repl/` 的新目录。这将把 REPL 放在它自己的 Rust 模块中。在 `repl/` 内部，创建一个名为 `mod.rs` 的文件。这是我们将定义我们结构的地方。

### REPL 结构

首先，我们需要导入一些东西：

```rust
use std;
use std::io;
use std::io::Write;
use vm::VM;
```

```rust
/// 汇编语言 REPL 的核心结构
pub struct REPL {
    command_buffer: Vec<String>,
    // REPL 将用来执行代码的虚拟机
    vm: VM,
}
```

到目前为止很简单！我们的 `impl` 块和典型的 `new()` 函数看起来像：

```rust
impl REPL {
    /// 创建并返回一个新的汇编 REPL
    pub fn new() -> REPL {
        REPL {
            vm: VM::new(),
            command_buffer: vec![]
        }
    }
```

### 命令缓冲区 (Command Buffer)

我们为什么要保留执行命令的列表？这样用户就可以按上箭头键并看到他们运行的内容。试试在 Python REPL 中操作！

### 虚拟机

REPL 结构将有一个它用来执行代码的虚拟机。这是因为 REPL 应该是首先创建的和最后销毁的。

### 另一个循环……

我们的无限循环位于 `impl` 块中的一个公共函数中。在 `main.rs` 中，我们将实例化 REPL 然后调用这个函数。它看起来像：

```rust
pub fn run(&mut self) {
    println!("Welcome to Iridium! Let's be productive!");
    loop {
        // 这将分配一个新的字符串，用来存储每次迭代用户输入的内容。
        // TODO: 弄清楚如何在外面创建这个并在每次迭代中重用它
        let mut buffer = String::new();

        // 阻塞调用，直到用户输入命令
        let stdin = io::stdin();

        // 令人烦恼的是，`print!` 不像 `println!` 那样自动刷新 stdout，所以我们
        // 必须在那里手动刷新，以便用户能看到我们的 `>>>` 提示。
        print!(">>> ");
        io::stdout().flush().expect("Unable to flush stdout");

        // 这里我们将查看用户给我们的字符串。
        stdin.read_line(&mut buffer).expect("Unable to read line from user");
        let buffer = buffer.trim();
        match buffer {
            ".quit" => {
                println!("Farewell! Have a great day!");
                std::process::exit(0);
            },
            _ => {
                println!("Invalid input");
            }
        }
    }
}
```

上面的循环是我们 REPL 无限循环的骨架。

#### 刷新 stdout

我们希望像 Python REPL 那样在所有输出前加上 `>>>`，但我们不想在结尾添加换行符，`println!` 会这样做。`print!` 不会添加换行符，但它也不刷新 stdout，所以用户不会看到 `>>>`。在上面的代码中，我们手动刷新它。

#### main.rs

现在让我们去 `src/main.rs` 并将其连接到我们的 REPL。首先，添加：

```rust
pub mod repl;
```

| | |
|---|---|
| 重要 | 如果你不记得 Rust 中模块是如何工作的，可以查看这个。|

然后在我们的 `main` 函数中：

```rust
fn main() {
    let mut repl = repl::REPL::new();
    repl.run();
}
```

一旦你有了这个，你应该能够输入 `cargo run` 并看到它在行动。输入 `.quit` 将退出 REPL。

```sh
$ cargo run
   Compiling iridium_part_02 v0.1.0 (file:///Users/fletcher/Projects/iridium-book)
warning: field is never used: `command_buffer`
 --> src/repl/mod.rs:8:5
  |
8 |     command_buffer: Vec<String>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `vm`
 --> src/repl/mod.rs:9:5
  |
9 |     vm: VM,
  |     ^^^^^^

    Finished dev [unoptimized + debuginfo] target(s) in 0.44s
     Running `target/debug/iridium_part_02`
Welcome to Iridium! Let's be productive!
>>> .quit
Farewell! Have a great day!
```

不用担心未使用变量的警告。我们很快就会解决这个问题。=)

### 命令历史

既然我们在这里，让我们实现存储用户输入的历史。我们已经初始化了我们的 `Vec`，所以问题在于将每个输入的副本追加到其中，如下所示：

```rust
stdin.read_line(&mut buffer).expect("Unable to read line from user");
let buffer = buffer.trim();

// 这是我们添加的用于存储每个命令副本的行
self.command_buffer.push(buffer.to_string());

match buffer {
```

我们如何使用命令缓冲区？现在，让我们添加另一个命令 `.history`，它将打印出以前的命令。稍后，我们将使其更加复杂。

#### 历史命令

在 `src/repl/mod.rs` 中的 `.quit` 命令下方，添加这个：

```rust
".history" => {
    for command in &self.command_buffer {
        println!("{}", command);
    }
},
```

现在测试它：

```sh
$ cargo run
   Compiling iridium_part_02 v0.1.0 (file:///Users/fletcher/Projects/iridium-book)
Welcome to Iridium! Let's be productive!
>>> invalid
Invalid input
>>> .hello
Invalid input
>>> .history
invalid
.hello
.history
>>>
```

## 结束

这就结束了本篇文章！在下一篇文章中，我们将为我们的 REPL 添加输入字节码和执行它。

### 原文链接及作者信息

- 原文链接：[The REPL - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-06/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- REPL（Read, Evaluate, and Print Loop）：读取、评估和打印循环，也称为交互式解释器。
- 汇编语言（Assembly Language）：一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- Rust 枚举（Rust Enums）：Rust 语言中的一种类型，用于表示一组相关的值。

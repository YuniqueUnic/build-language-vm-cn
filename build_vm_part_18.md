# 18-进程标识符 - 你想构建一个语言虚拟机吗？

- [18-进程标识符 - 你想构建一个语言虚拟机吗？](#18-进程标识符---你想构建一个语言虚拟机吗)
    - [进程 ID（PIDs）](#进程-idpids)
        - [Iridium 虚拟机的标识符 (`Identifiers`)](#iridium-虚拟机的标识符-identifiers)
        - [进程 (Processes)](#进程-processes)
            - [跟踪事件](#跟踪事件)
        - [快要完成了](#快要完成了)
        - [应用程序 ID](#应用程序-id)
            - [更新 VMEvent](#更新-vmevent)
    - [临时解决方案 (Hackety Hack)](#临时解决方案-hackety-hack)
        - [CLI](#cli)
        - [REPL](#repl)
    - [结束](#结束)
        - [编码风格](#编码风格)
        - [俚语\&典故](#俚语典故)
        - [专有名词及注释](#专有名词及注释)

大家好！在本教程中，我们将为 Iridium 虚拟机添加进程 ID（PID）跟踪功能。请确保您从以下标签开始：<https://gitlab.com/subnetzero/iridium/tags/0.0.17>。

## 进程 ID（PIDs）

我们需要有两个组件来唯一标识：

1. Iridium 虚拟机

2. 这些虚拟机运行的进程

这确实引出了一个更基本的问题。一个`虚拟机`是长期存在的，还是短期存在的？我们应该为每个我们想要运行的应用程序创建一个拥有自己的寄存器和堆的虚拟机吗？我们应该创建一个虚拟机池 (`pool`)，每个虚拟机都在自己的线程中，等待运行我们加载的任何应用程序吗？

| | |
|---|---|
| 警告 | 重用 (`re-using`) 虚拟机时存在安全考虑。如果我们这样做，我们必须确保在允许另一个应用程序访问之前将寄存器和堆清零。否则，应用程序可以从以前的虚拟机中读取数据。|

### Iridium 虚拟机的标识符 (`Identifiers`)

为此，我们将使用一个随机的 UUID。在创建时，虚拟机将为自己生成一个随机标识符。这将适用于我们最终如何处理多个虚拟机。

对于生成随机 UUID，这个 [UUID crate](https://github.com/uuid-rs/uuid) 非常方便。将其添加到您的`Cargo.toml`中，并且不要忘记在`main.rs`中添加`extern crate uuid;`。

| | |
|---|---|
| 注意 | 由于 UUID 将是随机的，我们需要启用 v4 特性：`uuid = { version = "0.7", features = ["v4"] }`。|

现在在`src/vm.rs`中，让我们为我们的虚拟机添加一个字段：

```rust
/// 将执行字节码的虚拟机结构
#[derive(Default, Clone)]
pub struct VM {
    /// 模拟硬件寄存器的数组
    pub registers: [i32; 32],
    /// 跟踪正在执行的字节的程序计数器
    pc: usize,
    /// 正在运行的程序的字节码
    pub program: Vec<u8>,
    /// 用于堆内存
    heap: Vec<u8>,
    /// 包含模除操作的余数
    remainder: usize,
    /// 包含最后一次比较操作的结果
    equal_flag: bool,
    /// 包含只读部分数据
    ro_data: Vec<u8>,
    /// 用于标识这个虚拟机的唯一随机生成的 UUID
    id: Uuid,
}
```

以及我们在`impl VM`中的构建函数：

```rust
/// 创建并返回一个新的虚拟机
pub fn new() -> VM {
    VM {
        registers: [0; 32],
        program: vec![],
        ro_data: vec![],
        heap: vec![],
        pc: 0,
        remainder: 0,
        equal_flag: false,
        id: Uuid::new_v4()
    }
}
```

（我不会为它编写测试，因为这不可能失败）

### 进程 (Processes)

我们真正想要的只是一个事件日志：“应用程序 X 在<时间戳>运行，并在<时间戳>终止，退出代码为<代码>”。

理论上，一个长时间运行的虚拟机可能会重用 ID，这可能会造成混淆 (`confusing`)。让我们也给每个应用程序一个随机 UUID。

回到`vm.rs`并添加这个：

```rust
use chrono::prelude::*;

#[derive(Clone, Debug)]
pub enum VMEventType {
    Start,
    GracefulStop,
    Crash
}

#[derive(Clone, Debug)]
pub struct VMEvent {
    event: VMEventType,
    at: DateTime<Utc>
}
```

请留意：chrono 库的引入：<https://github.com/chronotope/chrono>。借助它，我们可以更加便捷地操作日期与时间。

| | |
|---|---|
| 注意 | 是的，所有的时间都将使用 UTC。我现在对所有在日志中使用时区的人皱眉头 (`scowling`)。|

将 [chrono 包](https://github.com/chronotope/chrono)添加到您的 Cargo.toml 中，以及所有其他内容。

#### 跟踪事件

目前，我们将给虚拟机一个 VMEvents 列表，我们将不断追加到这个列表中。

```rust
/// 将要执行字节码的虚拟机结构
#[derive(Default, Clone)]
pub struct VM {
    /// ...
    // 我移除了其他字段，因为我们已经看过了
    events: Vec<VMEvent>
}

```

以及… …

```rust
pub fn new() -> VM {
    VM {
        // ...
        // 我移除了其他字段，因为我们已经看过了
        events: Vec::new()
    }
}
```

### 快要完成了

让我们修改虚拟机，在`run()`函数开始、停止或崩溃时添加一个事件：

```rust
/// 将执行包裹在一个循环中，以便它会一直运行，直到完成或执行指令时出错。
pub fn run(&mut self) -> u32 {
    self.events.push(VMEvent{event: VMEventType::Start, at: Utc::now()});
    // TODO: 这里应该设置自定义错误
    if !self.verify_header() {
        self.events.push(VMEvent{event: VMEventType::Crash, at: Utc::now()});
        println!("头部不正确");
        return 1;
    }
    // 如果头部有效，我们需要将 pc 更改为位 65。
    self.pc = 64;
    let mut is_done = false;
    while !is_done {
        is_done = self.execute_instruction();
    }
    self.events.push(VMEvent{event: VMEventType::Stop, at: Utc::now()});
    0
}
```

注意我们假设只要 while 循环结束，应用程序就正常终止了。这是因为`execute_instruction`返回一个 bool，而不是一个整数 (`integer`)。叹息 (`Sigh`)。

让我们改变它。这将是有点痛苦 (`painful`) 的，但以后会更痛苦。

首先，我们必须改变返回值：

```rust
fn execute_instruction(&mut self) -> u32
```

然后在检查 pc 是否超出程序长度时：

```rust
if self.pc >= self.program.len() {
    return 1;
}
```

对于 HLT 和 IGL 代码：

```rust
Opcode::HLT => {
    println!("遇到 HLT");
    return 0;
}
Opcode::IGL => {
    println!("遇到非法指令");
    return 1;
}

```

以及最后一行，我们在 opcode 返回或应用程序完成时返回 false：

```rust
fn execute_instruction(&mut self) -> u32 {
    if self.pc >= self.program.len() {
        return 1;
    }
    match self.decode_opcode() {
        Opcode::LOAD => {
            let register = self.next_8_bits() as usize;
            let number = u32::from(self.next_16_bits());
            self.registers[register] = number as i32;
        }
        // <snip a lot of other opcodes>
        // 内容被截断了，很多其他的操作码被省略了。
    };
    0
}

```

现在我们去改变`run`函数：

```rust
pub fn run(&mut self) -> u32 {
    self.events.push(VMEvent{event: VMEventType::Start, at: Utc::now()});
    // TODO: 这里应该设置自定义错误
    if !self.verify_header() {
        self.events.push(VMEvent{event: VMEventType::Crash{code: 1}, at: Utc::now()});
        println!("头部不正确");
        return 1;
    }
    // 如果头部有效，我们需要将 pc 更改为位 65。
    self.pc = 64;
    let mut is_done = None;
    while is_done.is_none() {
        is_done = self.execute_instruction();
    }
    self.events.push(VMEvent{event: VMEventType::GracefulStop{code: is_done.unwrap()}, at: Utc::now()});
    0
}
```

糟糕 (`Crap`)，问题在于我们错误地将返回码 0 视为应用程序完成的信号，但现在，某些指令（例如 HLT）也会返回 0。这就导致程序即使在不应继续时也会继续运行。

这是否意味着 HLT 应该返回一个大于 0 的值？老实说，我不确定。但我确实知道我不想违背 *nix 的惯例 (`convention`)，即 0 表示正常，大于 0 表示某种错误……

哦，对了，Rust 有一个美妙的东西叫做 Option<_>……哈哈，Option。让我们尝试使用一个 Option 作为继续执行的信号。

| | |
|---|---|
| 注意 | 我一边写代码一边记录这些思考过程，所以你可以看到我的思路。让我们尝试在 vm.rs 中的 run 函数实现这个想法。|

让我们在 `vm.rs` 中尝试 run 函数：

```rust
/// Wraps execution in a loop so it will continue to run until done or there is an error
/// executing instructions.
pub fn run(&mut self) -> u32 {
    self.events.push(VMEvent{event: VMEventType::Start, at: Utc::now()});
    // TODO: Should setup custom errors here
    if !self.verify_header() {
        self.events.push(VMEvent{event: VMEventType::Crash{code: 1}, at: Utc::now()});
        println!("Header was incorrect");
        return 1;
    }
    // If the header is valid, we need to change the PC to be at bit 65.
    self.pc = 64;
    let mut is_done = None;
    while is_done.is_none() {
        is_done = self.execute_instruction();
    }
    self.events.push(VMEvent{event: VMEventType::GracefulStop{code: is_done.unwrap()}, at: Utc::now()});
    0
}
```

注意我们在添加停止事件时必须解包`is_done`。

然后在`execute_instruction`函数中：

```rust
fn execute_instruction(&mut self) -> Option<u32> {
    if self.pc >= self.program.len() {
        return Some(1);
    }
}
```

注意签名 (`signature`) 的返回类型也改变了，别忘了修复 HLT 和 IGL 的 opcode。

最后，我们的`run`函数的结尾：

```rust
pub fn run(&mut self) -> u32 {
    // <snip>
    None
}

```

运行`cargo test`确保我们没有破坏任何东西……

```sh
测试结果：ok。44 通过；0 失败；0 忽略(ignored)；0 测量(measured)；0 筛选(filtered out)
```

耶！(`Yay!`)

### 应用程序 ID

目前，我们只是每次运行应用程序时使用一个新的虚拟机。这使得虚拟机 ID 与应用程序 ID 相同。我们可能想要考虑构建一个稍微 (`slightly`) 更抽象的程序形式，以便我们可以附加额外的 (`additional`) 信息。

#### 更新 VMEvent

让我们更新 VMEvent 以包含一个 id 字段：

```rust
#[derive(Clone, Debug)]
pub struct VMEvent {
    event: VMEventType,
    at: DateTime<Utc>,
    application_id: Uuid
}

```

然后在我们生成事件的三个地方，从虚拟机 id 克隆它。我们的`run`函数现在应该看起来像这样：

```rust
/// 将执行包裹在一个循环中，以便它会一直运行，直到完成或执行指令时出错。
pub fn run(&mut self) -> u32 {
    self.events.push(
        VMEvent{
            event: VMEventType::Start,
            at: Utc::now(),
            application_id: self.id.clone()
        }
    );
    // TODO: 这里应该设置自定义错误
    if !self.verify_header() {
        self.events.push(
            VMEvent{
                event: VMEventType::Crash{
                    code: 1
                },
                at: Utc::now(),
                application_id: self.id.clone()
            }
        );
        println!("头部不正确");
        return 1;
    }
    // 如果头部有效，我们需要将 pc 更改为位 65。
    self.pc = 64;
    let mut is_done = None;
    while is_done.is_none() {
        is_done = self.execute_instruction();
    }
    self.events.push(
        VMEvent{
            event: VMEventType::GracefulStop{
                code: is_done.unwrap()},
                at: Utc::now(),
                application_id: self.id.clone()
        }
    );
    0
}

```

并且...糟糕 (damnit)! 结果还是老样子！我们依然从 `run` 函数返回 1 或 0。所以我们的精美事件集合消失 (`vanish`) 了。

唉 (`Sigh`)，好吧，让我们修改 `run` 函数，让它返回一个事件列表，我们将 1 和 0 的返回值改为返回整个事件向量 (`Vector of events`)。最终的 `run` 函数应该看起来像这样：

```rust
pub fn run(&mut self) -> Vec<VMEvent> {
    self.events.push(
        VMEvent{
            event: VMEventType::Start,
            at: Utc::now(),
            application_id: self.id.clone()
        }
    );
    // TODO: 这里应该设置自定义错误
    if !self.verify_header() {
        self.events.push(
            VMEvent{
                event: VMEventType::Crash{
                    code: 1
                },
                at: Utc::now(),
                application_id: self.id.clone()
            }
        );
        println!("头部不正确");
        return self.events.clone();
    }
    // 如果头部有效，我们需要将 pc 更改为位 65。
    self.pc = 64;
    let mut is_done = None;
    while is_done.is_none() {
        is_done = self.execute_instruction();
    }
    self.events.push(
        VMEvent{
            event: VMEventType::GracefulStop{
                code: is_done.unwrap()},
                at: Utc::now(),
                application_id: self.id.clone()
        }
    );
    self.events.clone()
}
```

`cargo test`和：

```sh
error[E0308]: mismatched types
  --> src/scheduler/mod.rs:21:7
   |
20 |       pub fn get_thread(&mut self, mut vm: VM) -> thread::JoinHandle<u32> {
   |                                                   ----------------------- expected `std::thread::JoinHandle<u32>` because of return type
21 | /       thread::spawn(move || {
22 | |           vm.run()
23 | |       })
   | |________^ expected u32, found struct `std::vec::Vec`
   |
   = note: expected type `std::thread::JoinHandle<u32>`
              found type `std::thread::JoinHandle<std::vec::Vec<vm::VMEvent>>`
```

好的，编译器。我们去`src/scheduler/mod.rs`。添加一个导入：

```rust
use vm::{VM, VMEvent};
```

然后改变`get_thread`的签名：

```rust
/// 采用一个虚拟机并在后台线程中运行它
pub fn get_thread(&mut self, mut vm: VM) -> thread::JoinHandle<Vec<VMEvent>> {
  thread::spawn(move || {
      vm.run()
  })
}
```

`cargo test`说一切都很好，编译器没有对我们大喊大叫 (`yelling`)……我们完成了吗？

哈。不，当然没有！我们仍然没有向用户展示结果。

## 临时解决方案 (Hackety Hack)

目前，我们只是在调用 run 时打印事件日志。我们需要在两个地方这样做：

1. 当用户从 CLI 运行程序时，例如，`iridium myfile.iasm`

2. 当用户通过 REPL 运行程序时

我们将在稍后对其进行格式化，使其看起来更好，但这篇文章已经有 2033 个词了。

让我们按顺序解决 (`tackle`) 它们。

### CLI

在`main.rs`中，我们有这个部分：

```rust
let program = asm.assemble(&program);
match program {
    Ok(p) => {
        vm.add_bytes(p);
        vm.run();
        std::process::exit(0);
    },
    Err(_e) => {

    }
}
```

让我们将`run`的输出赋值给一个变量，然后进行调试打印：

```rust
match program {
    Ok(p) => {
        vm.add_bytes(p);
        let events = vm.run();
        println!("虚拟机事件");
        println!("--------------------------");
        for event in &events {
            println!("{:#?}", event);
        };
        std::process::exit(0);
    },
    Err(_e) => {

    }
```

### REPL

我将把在 REPL 中显示的任务留给你。你可以在 [GitLab](https://gitlab.com/subnetzero/iridium/tags/0.0.18) 上看到我的。

## 结束

我们将在这里结束这个教程，尽管我想做一个观察。

### 编码风格

我在 Rust 中的编码风格对于这样一个严格的语言来说出奇地自由。在编写 Rust 代码时，我的生活目标变成了安抚编译器。只要我能这样做，我编码的东西通常就会像我认为的那样工作。

我们下次教程见！

### 俚语&典故

1. `A Faustian Bargain`：浮士德式的交易 (A Faustian Bargain)，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

2. `Zergling`: 是来自于游戏《星际争霸》（StarCraft）中的一个单位，属于 Zerg 种族的基本战斗单位。在这里，作者可能是以幽默的方式，将线程比作是快速、大量生产的小型战斗单位，以此来形象地描述线程的创建和管理工作。在翻译时，可以保留原文`zergling`，因为它已经成为了游戏文化中的一个专有名词，或者可以解释性地翻译为`快速生成的战斗单位`或`轻量级线程`。

### 专有名词及注释

1. Snip: 的完整拼写就是 snip。这个单词通常用于表示剪断或截取的动作。在计算机领域，它有时被用来指代对文本或数据的一部分进行截取或省略的操作。
2. Hackety Hack: 这个短语通常是指一种快速而可能不太优雅的编程方式，用来解决一个问题或完成一个任务，尤其是在需要迅速实现功能或者修复紧急问题时。这个词组可以用来描述以下几种情况：
    1. 编程风格：指编写代码时采用了一些快捷方法，这些方法可能不是最规范的，但能够迅速解决问题。
    2. 临时解决方案：可能是指一个临时的修复或者是一个“凑合用”的代码片段，它可能不是长期维护的最好选择。
    3. 破解行为：在某些情况下，它也可能指的是对软件或系统进行的一些非官方的修改，以达到某种目的。


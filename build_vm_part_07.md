# 07-更高级的 REPL - 你想构建一个语言虚拟机吗？

- [07-更高级的 REPL - 你想构建一个语言虚拟机吗？](#07-更高级的-repl---你想构建一个语言虚拟机吗)
    - [十六进制 (Hexadecimal)](#十六进制-hexadecimal)
        - [示例](#示例)
            - [获取操作码](#获取操作码)
            - [获取第一个操作数](#获取第一个操作数)
            - [获取第二个和第三个操作数](#获取第二个和第三个操作数)
            - [汇总](#汇总)
    - [扩展 REPL](#扩展-repl)
        - [.program](#program)
        - [.registers](#registers)
        - [输入十六进制](#输入十六进制)
        - [它有效吗？](#它有效吗)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)


我们的当前 REPL 功能不多，所以让我们改进一下。在这篇文章中，我们将添加一些命令来查看程序字节码和寄存器及其内容，以及实际执行输入的十六进制代码。

| | |
|---|---|
| 重要 | 你需要对十六进制有基本的了解才能进行这部分。我将在此处介绍一些部分，但你可能需要阅读[更详细的解释](https://simple.wikipedia.org/wiki/Hexadecimal_numeral_system)。|

## 十六进制 (Hexadecimal)

我们使用的是所谓的十进制编号系统，即基数为 10。为什么呢？可能是因为我们有 10 个手指。我们大多数人都习惯使用 0-9 的数字。

十六进制编号系统，即基数为 16，使用 0-9 的数字和 A-F 的字母。这意味着一个 _位_ 可以表示 0-16。

这对我们来说很方便，因为一个十六进制数字可以表示 4 位，所以 2 个可以表示 8 位，或 1 个字节。这意味着我们只需要 8 个数字就可以表示我们的一条指令。

### 示例

我们将构建一个 `LOAD` 指令，将数字 1000 载入寄存器 1。

#### 获取操作码

如果你回顾 `instruction.rs`，你可以看到我们的 `LOAD` 指令使用数字 0。在十六进制中的表示是：`00`。

#### 获取第一个操作数

使用 `LOAD` 指令时，我们的第一个操作数是我们想要存储数字的寄存器编号。我们想要使用寄存器 12，十六进制中表示为 `C`。由于我们需要使用两个字节，我们这样填充：`0C`。

#### 获取第二个和第三个操作数

最后两个字节用于存储数字。记住，目前我们限制在 2^16 以内。我们存储的数字是 1000，十六进制中表示为 `03 E8`。

#### 汇总

我们完整的指令看起来像这样：`00 0C 03 E8`。现在，让我们为 REPL 添加一些命令。

## 扩展 REPL

我们将向 `REPL` 添加两个更多命令：

1. `.program`
2. `.registers`

### .program

这将打印出虚拟机程序向量中的整个字节码。实现看起来像这样：

```rust
".program" => {
    println!("正在列出虚拟机程序向量中的指令：");
    for instruction in &self.vm.program {
        println!("{}", instruction);
    }
    println!("程序列表结束");
},
```

| | |
|---|---|
| 重要 | 要能够访问虚拟机结构的程序和寄存器字段，你需要在它们前面添加 `pub`!|

### .registers

这将列出所有 32 个寄存器及其当前值。这对于验证我们的指令是否按预期工作非常有用：

```rust
".registers" => {
    println!("正在列出所有寄存器及其内容：");
    println!("{:#?}", self.vm.registers);
    println!("寄存器列表结束")
},
```

### 输入十六进制

我们的最后一项任务是接受用户输入，由 8 个十六进制数字组成，分成 2 个一组。为此，我们将向我们的 `REPL` 添加一个名为 `parse_hex` 的新函数：

```rust
/// 接受一个十六进制字符串 WITHOUT a leading `0x` 并返回一个 u8 的 Vec
/// 示例对于一个 LOAD 命令：00 01 03 E8
fn parse_hex(&mut self, i: &str) -> Result<Vec<u8>, ParseIntError>{
    let split = i.split(" ").collect::<Vec<&str>>();
    let mut results: Vec<u8> = vec![];
    for hex_string in split {
        let byte = u8::from_str_radix(&hex_string, 16);
        match byte {
            Ok(result) => {
                results.push(result);
            },
            Err(e) => {
                return Err(e);
            }
        }
    }
    Ok(results)
}
```

这个函数的目的是我们可以在 REPL 中直接输入字节码并执行它。使用十六进制作为输入让我们只需要输入 8 个字符，而不是 32 个。接下来，我们需要改变 REPL 处理输入匹配的“其他”部分的方式。目前，它检查一些命令并丢弃其他所有内容：

```rust
match buffer {
    ".quit" => {
        println!("再见！祝你有美好的一天！");
        std::process::exit(0);
    },
    _ => {
        println!("无效输入");
    }
}
```

我们将尝试解析十六进制并将其交给虚拟机运行，而不是打印无效输入：

```rust
_ => {
    let results = self.parse_hex(buffer);
    match results {
        Ok(bytes) => {
            for byte in bytes {
                self.vm.add_byte(byte)
            }
        },
        Err(_e) => {
            println!("无法解码十六进制字符串。请输入 4 组 2 个十六进制字符。")
        }
    };
    self.vm.run_once();
}
```

### 它有效吗？

让我们找出答案！

```sh
欢迎使用 Iridium！让我们提高效率！
>>> .registers
正在列出所有寄存器及其内容：
[
    0,
    0,
    <snip>,
]
寄存器列表结束
>>> 00 01 03 E8
>>> .registers
正在列出所有寄存器及其内容：
[
    0,
    1000,
    <snip>,
]
寄存器列表结束
>>>
```

很酷，对吧？我们输入了十六进制字符，我们的虚拟机执行了它们！

现在，用十六进制进行所有编程的想法是可怕的；老一辈的程序员将所有代码都必须用十六进制编写的时期称为《糟糕的时代》。但是，这个项目的一个重点就是体验每一层！

## 结束

这篇文章就到这里。在下一篇文章中，我们将开始构建一个汇编器！

### 原文链接及作者信息

- 原文链接：[REPL and Code Execution - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-07/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- REPL (Read Evaluate Print Loop): 读取、评估和打印循环，也称为交互式解释器。
- 十六进制 (Hexadecimal): 一种基数为 16 的计数系统，使用数字 0-9 和字母 A-F 表示值。
- 汇编语言 (Assembly Language): 一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- Rust 枚举 (Rust Enums): Rust 语言中的一种类型，用于表示一组相关的值。

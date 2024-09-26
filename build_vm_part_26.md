# 26-添加浮点数支持 - 你想构建一个语言虚拟机吗？

- [26-添加浮点数支持 - 你想构建一个语言虚拟机吗？](#26-添加浮点数支持---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [整数](#整数)
        - [浮点数](#浮点数)
        - [CPU 表示](#cpu-表示)
    - [寄存器](#寄存器)
        - [选项 1](#选项-1)
        - [选项 2](#选项-2)
        - [选项 3](#选项-3)
        - [获胜者是……](#获胜者是)
    - [实现](#实现)
        - [向虚拟机添加 f64 寄存器](#向虚拟机添加-f64-寄存器)
        - [新指令](#新指令)
        - [一个陷阱 (Gotcha)](#一个陷阱-gotcha)
    - [汇编器](#汇编器)
        - [测试](#测试)
    - [\> 2^16](#-216)
        - [位位移](#位位移)
            - [位位移的变体](#位位移的变体)
        - [快速位移](#快速位移)
            - [SHL 实现](#shl-实现)
            - [还有一个测试……](#还有一个测试)
        - [关于浮点数的最后说明](#关于浮点数的最后说明)
    - [结束](#结束)
        - [专有名词及注释](#专有名词及注释)

## 引言

大家好！在本教程中，我们将升级我们的虚拟机以支持浮点数。在之前的教程中，我们为 Palladium 语言添加了支持，所以我们也需要在虚拟机中支持它。在我们开始实现之前，让我们快速回顾一下数字。

### 整数

`Integers`

这些也被称为 _整数_。例如 1、2、3、4 等。它也包括负数：-1、-2、-3 等。0 也是一个整数。如果我们从货币的角度来看这些数字，我们可以看到一个问题……你如何表示 1.50 美元？你不能，你要么有 1 美元要么有 2 美元。

### 浮点数

`Floating Points`

这些数字包含小数点，当需要更高的精度时使用。我们可以表示像 1.50 美元这样的分数。

### CPU 表示

那我们为什么不到处使用浮点数呢？嗯，我们本可以。实际上，一些语言确实这样做了。那么为什么大多数语言要区分 (`distinguish`) 它们呢？让我们来看看计算机内部是如何表示整数的。

在二进制中，每个比特是 1 或 0。我们将 8 个比特组成一个称为字节的东西。我们可以使用这种方式来表示整数，而不会有问题：

```rust
[0, 0, 0, 0, 0, 0, 0, 0] // 1 byte = 8 bits
```

整数 1 表示为 `[0, 0, 0, 0, 0, 0, 0, 1]`，以此类推。这就是我们的十进制系统的工作方式。

| | |
|---|---|
| 注 | 你可能会想知道我们如何表示负整数。好问题！一个常见的方法是使用一个比特作为 _符号_ 比特。在我们 8 位的例子中，我们可以说最左边的比特是我们的符号比特。如果它是 0，数字是正数。如果它是 1，它是负数。整数 -1 可以表示为：`[1, 0, 0, 0, 0, 0, 0, 1]`。这就是你在编程语言中看到的 _有符号_ (_`signed`_) 和 _无符号_ (_`unsigned`_) 数字之间的区别。</br></br>使用 8 位，没有符号比特，我们可以表示 2^8，或 0-256。使用 8 位，有一个符号比特，我们可以表示 2^7，包括正数和负数。所以我们的范围变成了 -128 到 127。|

> 原文为 -128 to 128 是不正确的，正确的是 -128 to 127, 本文已更正错误。

如果我们想用二进制表示浮点数，我们必须做更多的工作。我不会在这里详细介绍，因为网上有很多优秀的解释。总结来说，与整数相比，CPU 处理浮点数需要更多的工作。

## 寄存器

好的，让我们开始实现吧！Iridium 有 32 个 i32 寄存器。i32 意味着 32 位整数。我们不能直接把一个 64 位的浮点数塞进去，对吧？我们有几个选择：

> 1. Duplicate the existing register design, but use f64 instead of i32. This is similar to a chip manufacturer adding more hardware registers.
> 2. Change the existing registers to hold an array of u8s.
> 3. Tinkering with unsafe raw pointers to cast between types.

1. 复制现有的寄存器设计，但使用 f64 而不是 i32。这类似于芯片制造商增加更多的硬件寄存器。

2. 将现有寄存器更改为持有 u8 数组。

3. 通过不安全的原始指针来转换类型。

### 选项 1

这个选项的优点是简单，但它也会导致很多代码重复。现在我们需要为 64 位寄存器准备单独的指令。如果有人试图将 32 位寄存器与 64 位寄存器相加怎么办？我们允许它们之间的转换吗？

### 选项 2

这个选项的优点是更通用的解决方案，但它也需要我们编写大量的代码来管理 u8。

### 选项 3

我们的主要 (`primary`) 目标之一是尽可能不使用不安全的 Rust。

### 获胜者是……

选项 1！它是最容易编写和理解的。它可能会让用户感到有点麻烦，但我们可以做一些工作来帮助解析器。

## 实现

我们将需要做的步骤总结如下：

> 1. Add an array of f64s to the VM
> 2. Add f64-bit versions of the LOAD, ADD, SUB, MUL, DIV and comparison operators
> 3. Teach our parser and assembler about them

1. 向虚拟机添加一个 f64 数组

2. 添加 LOAD, ADD, SUB, MUL, DIV 和比较运算符的 f64 版本

3. 教我们的解析器和汇编器了解它们

让我们开始吧！

### 向虚拟机添加 f64 寄存器

这是一项简单的工作。在 `src/vm.rs` 中，添加一个名为 `float_registers` 的字段。

```rust
pub struct VM {
    /// 模拟硬件寄存器的数组
    pub registers: [i32; 32],
    /// 模拟浮点硬件寄存器的数组
    pub float_registers: [f64; 32],
    /// ...
}

```

然后在 VM 构造函数中，我们需要初始化它：

```rust
impl VM {
    /// 创建并返回一个新的虚拟机
    pub fn new() -> VM {
        VM {
            registers: [0; 32],
            float_registers: [0.0; 32],
    // ...
  }
}

```

注意我们将它们默认为 `0.0`。这是因为它们是 f64。

### 新指令

我们需要一堆 (`a bunch of`) 新指令：

01. LOADF64

02. ADDF64

03. SUBF64

04. MULF64

05. DIVF64

06. EQF64

07. NEQF64

08. GTF64

09. GTEF64

10. LTF64

11. LTEF64

在 `src/instructions.rs` 中，将这些添加到 `Opcodes` 列表中，为 `From<u8>` impl 添加，以及为 `From<CompleteStr<'a>>` 为 `Opcode` trait 添加。

但等等，还有更多！

我们还需要在 `src/vm.rs` 中为每个新指令添加一个匹配分支。`ADDF64` 的实现如下：

```rust
Opcode::ADDF64 => {
    let register1 = self.float_registers[self.next_8_bits() as usize];
    let register2 = self.float_registers[self.next_8_bits() as usize];
    self.float_registers[self.next_8_bits() as usize] = register1 + register2;
},

```

注意它与 `ADD` 指令相同，只是我们访问了 float_registers 数组。

### 一个陷阱 (Gotcha)

让我们来看一下 32 位指令的 `EQ` 操作码：

```rust
Opcode::EQ => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    self.equal_flag = register1 == register2;
    self.next_8_bits();
}
```

`==` 对于比较整数来说很好，因为它们可以在二进制中精确表示。然而，两个浮点数可能会有非常小的差异。例如，假设我们有 `1.1003` 和 `1.1004`。我们可能不在乎一个比另一个高出 1/10000，而只在乎它们是否在十分之一以内相等。这种误差幅度通常被称为 _epsilon_。Rust 通过 `use std::f64::EPSILON;` 使其可用。

看看我们的 `EQ64` 指令：

```rust
Opcode::EQF64 => {
    let register1 = self.float_registers[self.next_8_bits() as usize];
    let register2 = self.float_registers[self.next_8_bits() as usize];
    self.equal_flag = (register1 - register2).abs() < EPSILON;
    self.next_8_bits();
}
```

我们检查两者之间的差异是否在 epsilon 之内，如果是，则认为它们相等。所有 64 位比较指令都必须使用 epsilon。尝试一下，如果你遇到困难，可以查看源代码。

## 汇编器

我们需要更新的最后一件事是我们的汇编器，教它识别浮点数。在 `src/operand_parsers.rs` 中，添加这个：

```rust
named!(float_operand<CompleteStr, Token>,
    ws!(
        do_parse!(
            sign: opt!(tag!("-")) >>
            left_nums: digit >>
            tag!(".") >>
            right_nums: digit >>
            (
                {
                    let mut tmp = String::from("");
                    if sign.is_some() {
                        tmp.push_str("-");
                    }
                    tmp.push_str(&left_nums.to_string());
                    tmp.push_str(".");
                    tmp.push_str(&right_nums.to_string());
                    let converted = tmp.parse::<f64>().unwrap();
                    Token::Factor{ value: Box::new(Token::Float{value: converted}) }
                }
            )
        )
    )
);

```

这个解析器寻找：

> 1. `#`
> 2. An optional `-`
> 3. One or more digits
> 4. An `.`
> 5. One or more digits

1. `#`

2. 一个可选的 `-`

3. 一个或多个数字

4. 一个 `.`

5. 一个或多个数字

这样做让我们可以解析正浮点数和负浮点数，只要有一个 `.`。之后，我们在同一个文件中将新的浮点解析器添加到操作数解析器：

```rust
named!(pub operand<CompleteStr, Token>,
    alt!(
        integer_operand |
        float_operand |
        label_usage |
        register |
        irstring
    )
);

```

### 测试

让我们尝试一种更简明的 (`succinct`) 写测试方法：

```rust
#[test]
fn test_parse_float_operand() {
    vec!["#100.3", "#-100.3", "#1.0", "#0.0"].iter().map(|i| {
        assert_eq!(float_operand(CompleteStr(i)).is_ok(), true);
    });
}
```

仍然相当清晰但更紧凑。相同的模式也可以应用于其他测试。我会重构一些，但不会在我们的教程中介绍。

因为我们有一个问题。一个大（或者不那么大）的问题，这取决于你怎么看。

## \> 2^16

我们目前的固定宽度 (`fixed-width`) 32 位指令设计，我们失去了 8 位给操作码。在 `LOAD` 的情况下，我们又失去了另外 8 位来指定寄存器。这给我们只剩下 16 位来表示要加载到该寄存器中的数字。所以即使我们有 i32 和 f64 寄存器，我们也无法充分利用它们。我们需要一种方法让程序员输入大于 2^16 的数字。

实际上有很多方法可以做到这一点。一个常见的方法是结合位位移指令和加载。(`A common one is to combine bit-shifting instructions with loads.`)

### 位位移

`Bit Shifting`

假设我们想把一个 32 位的数字加载到一个 32 位的寄存器中。我们能做的是有一个指令将目标寄存器的前半部分设置为寄存器的前 16 位，然后一个指令将这些位移到右 16 位。

所以我们从：

```sh
[0,1,0,0,0,1,1,0,0,0,0,1,0,0,0,0 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
                                → 
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,1,1,0,0,0,0,1,0,0,0,0]
```

现在我们可以使用相同的指令再次设置前 16 位。

#### 位位移的变体

`A Variant on Bit Shifting`

我们甚至不需要移动位，如果我们有一个指令来同时设置前 16 位和最后 16 位。当汇编器看到用户想要加载一个大于 16 位的数字时，它可以自动输出设置寄存器半部分所需的两个指令。你经常会看到这种指令被称为 LU（Load Upper），LOADU 等。这些指令通常很有礼貌，会在加载之前将高位下移到低位。

我们将在下一个教程中实现这些。

### 快速位移

`A Quick Shift`

在我们结束本教程之前，让我们编写一个快速的位移指令。新指令是：

1. SHR - 将 16 位向右移 
    1. (`SHR - Shifts 16 bits to the right`)

2. SHL - 将 16 位向左移 
    1. (`SHL - Shifts 16 bits to the left`)

我不会详细介绍添加操作码的细节，除了在 `src/vm.rs` 中的实现。你应该已经知道了。=)

这些命令的语法将是：

- SHR $<reg_num>
- SHR $<reg_num> #\<number of bits\>

第二种形式更有抱负，但如果允许用户选择要移动的位数，那将是很好的。

#### SHL 实现

下面是 `SHL` 指令的实现。剩下的一个你可以实现，或者参考源代码，因为它几乎相同。

```rust
Opcode::SHL => {
    let reg_num = self.next_8_bits() as usize;
    let num_bits = match self.next_8_bits() {
        0 => { 16 },
        other => { other }
    };
    self.registers[reg_num] = self.registers[reg_num].wrapping_shl(num_bits.into());
}

```

这段代码：

> 1. Gets the register the user wants to shift
> 2. Gets the next 8 bits, which is how many bits they want to shift
> 3. If it is 0, it defaults to 16 bits
> 4. If it is some other number, it shifts that amount

1. 获取用户想要位移的寄存器

2. 获取接下来的 8 位，这是他们想要移动的位数

3. 如果它是 0，它默认为 16 位

4. 如果它是其他数字，它移动那个数量

`wrapping_shl` 是内置于整数类型（u64, i32 等）的一个函数。

#### 还有一个测试……

```rust
#[test]
fn test_shl_opcode() {
    let mut test_vm = VM::get_test_vm();
    test_vm.program = vec![33, 0, 0, 0];
    assert_eq!(5, test_vm.registers[0]);
    test_vm.run_once();
    assert_eq!(327680, test_vm.registers[0]);
}

```

如果我们打印出零号寄存器的二进制表示，我们可以看到变化：

```sh
bits: 0b000000000000000000000000000101  # 5
bits: 0b000000000001010000000000000000  # 32768
```

第一行是数字 `5` 的二进制表示。如果我们向左移动 16 位，数字就变成了 `32768`。酷，对吧？

| | |
|---|---|
| 注 | 你知道在整数中，你可以通过将整数向左移动一位来实现乘以 2 的效果吗？</br>你可以通过将其向相反方向 (右) 移动一位来实现除以 2 的效果。|

### 关于浮点数的最后说明

你可能想知道为什么没有针对浮点数的位移指令。这是因为浮点数在内部的表示方式。位被分成两部分，有效数字和指数。如果你四处移动位，你可能会通过将它们移动到不同的部分来改变位的实际 _含义_。

## 结束

耶，我们做到了！接下来，我们将制作一些指令来加载寄存器的上半部分或下半部分。

原文出处：[Adding Floating Point Support - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-26/)
作者名：Fletcher

### 专有名词及注释

- LU（Load Upper）和 LOADU 指令通常出现在汇编语言或者某些处理器的指令集中，它们用于加载内存中的数据到寄存器。这里的“Upper”指的是加载操作数的较高部分，而“Load Upper”通常是一个复合指令，它可能由多个操作组成，用于加载一个多字节数据的高位部分。
    - LOADU（Load Unaligned）是一个不同的概念，它指的是一种加载操作，可以加载非对齐的数据。在许多处理器架构中，为了提高内存访问的效率，通常要求数据的加载和存储操作必须按照特定的字节边界对齐。但是，LOADU 指令允许加载的内存地址不必与特定的边界对齐，这在处理非标准或动态大小的数据结构时非常有用。
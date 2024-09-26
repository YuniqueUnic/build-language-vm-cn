# 03-更多基础操作码 - 你想构建一个语言虚拟机吗？

- [03-更多基础操作码 - 你想构建一个语言虚拟机吗？](#03-更多基础操作码---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [LOAD](#load)
            - [分解难度](#分解难度)
            - [代码实现](#代码实现)
            - [短暂绕道 (Detour)](#短暂绕道-detour)
        - [ADD](#add)
    - [SUB、MUL 和 DIV](#submul-和-div)
        - [DIV](#div)
    - [接下来是什么？](#接下来是什么)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)


## 引言

目前，我们的虚拟机只能做一件事：停机。这固然是一个重要特性，但我们可能需要添加更多的操作码来执行加载、相加、乘法等操作。

### LOAD

是时候让那些偷懒的寄存器开始发挥作用了！在我们未来的汇编语言中，LOAD 指令看起来会像这样：

```assembly
LOAD $0 #500
```

这将指示我们的虚拟机将数字 500 载入寄存器 0。

#### 分解难度

我们有 32 位可以使用，其中 16 位已经被占用：

- 8 位用于操作码 (`opcode`)

- 8 位用于寄存器编号 (`register number`)

这留给我们 16 位来存储数字。目前我们关心的是无符号整数，所以我们可以存储在 `LOAD` 指令中的最大数字是 2^16，或 65,536。稍后，我们将需要一种方法来处理更大的数字，但目前，我们将数字保持在该数值以下。

| | |
|---|---|
| 注 | 我们的寄存器可以存储一个 i32，但将负数表示为 `u8` 需要稍后解释的一点说明。|

#### 代码实现

处理 LOAD 操作码我们需要采取的步骤如下：

1. 解码前 8 位并查看 LOAD
2. 解码接下来的 8 位并用它来获取寄存器
3. 解码接下来的 16 位（分为 2 个 `u8`）为整数
4. 将它们存储在寄存器 (`register`) 中

```rust
Opcode::LOAD => {
    let register = self.next_8_bits() as usize; // 我们将其转换为 usize，以便我们可以用它作为数组的索引
    let number = self.next_16_bits() as u16;
    self.registers[register] = number as i32; // 我们的寄存器是 i32，所以我们需要进行转换。我们稍后会介绍。
    continue; // 开始循环的另一个迭代。等待读取的下 8 位应该是一个操作码。
},
```

你会注意到一些额外的辅助函数，`next_8_bits` 和 `next_16_bits`。你可以像这样将它们添加到 VM 实现块中：

```rust
fn next_8_bits(&mut self) -> u8 {
    let result = self.program[self.pc];
    self.pc += 1;
    return result;
}

fn next_16_bits(&mut self) -> u16 {
    let result = ((self.program[self.pc] as u16) << 8) | self.program[self.pc + 1] as u16;
    self.pc += 2;
    return result;
}
```

这些是获取下 8 位或 16 位并增加程序计数器的便利函数。

这就是 LOAD 操作码的全部内容！让我们为它编写一个测试：

```rust
#[test]
fn test_load_opcode() {
  let mut test_vm = get_test_vm();
  test_vm.program = vec![0, 0, 1, 244]; // 请记住，这是我们用两个 u8 以小端格式表示 500 的方式
  test_vm.run();
  assert_eq!(test_vm.registers[0], 500);
}
```

#### 短暂绕道 (Detour)

在我们的测试中调用 `run()` 是脆弱且相当 hacky 的。让我们添加一个函数，让我们执行一条指令。我们可以复制并粘贴整个 run 函数并进行调整，但是嗯...(ewww)。我们可以将执行操作码的代码提取到一个函数中：

```rust
/// 只要可以执行指令就循环。
pub fn run(&mut self) {
    let mut is_done = false;
    while !is_done {
        is_done = self.execute_instruction();
    }
}

/// 执行一条指令。旨在允许对虚拟机进行更受控的执行
pub fn run_once(&mut self) {
    self.execute_instruction();
}

fn execute_instruction(&mut self) -> bool {
    if self.pc >= self.program.len() {
        return false;
    }
    match self.decode_opcode() {
        Opcode::LOAD => {
            let register = self.next_8_bits() as usize;
            let number = self.next_16_bits() as u32;
            self.registers[register] = number as i32;
        },
        Opcode::HLT => {
            println!("遇到 HLT");
            false
        },
    }
    true
}
```

| | |
|---|---|
| 注 | 这为每次 VM 迭代增加了一个额外的函数调用。在性能方面，这并不好。当我们开始基准测试时，我们会想要重新审视这一点。|

### ADD

现在让我们编写 ADD 指令。它的形式如下：`ADD $0 $1 $2`。前两个操作数是我们想要相加的寄存器的值，第三个寄存器是值将结束的地方。在我们的汇编语言中，如果我们想要加载两个数字并将它们相加，它可能看起来像这样：

```assembly
LOAD $0 #10
LOAD $1 #15
ADD $0 $1 $2
```

寄存器 2 将有值 25。
ADD 使用指令的所有 4 个字节，因此我们的代码如下：

```rust
Opcode::ADD => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    self.registers[self.next_8_bits() as usize] = register1 + register2;
},
```

## SUB、MUL 和 DIV

SUB 和 MUL 操作码与 ADD 相同，但除法不是。因为我们的寄存器持有 u32 值，我们不能存储除法的小数结果。因此，我们需要存储余数 (`remainder`)。你可能会记得学校的这个：`8 / 5 = 1 remainder 3`。

为了满足这一需求，我们将向我们的虚拟机添加另一个属性，称为 `remainder`。我们新的 VM 结构如下：

```rust
#[derive(Debug)]
pub struct VM {
    registers: [i32; 32],
    pc: usize,
    program: Vec<u8>,
    remainder: u32,
}
```

### DIV

当我们遇到 DIV 操作码时，我们要做的就是将其除以，将商数存储在寄存器中，将余数存储在虚拟机的 `remainder` 属性中。这使我们的 DIV 匹配分支有点不同：

```rust
Opcode::DIV => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    self.registers[self.next_8_bits() as usize] = register1 / register2;
    self.remainder = (register1 % register2) as u32;
},
```

| | |
|---|---|
| 注 | 你可能想知道 % 是什么。那是 Rust 中的模运算符，我们用它来获取余数。|

## 接下来是什么？

我们的虚拟机现在可以做一些数学运算了！你还想要什么？！

好吧，好的，我们应该可能还需要添加一些更多的特性。在下一篇文章中，我们将讨论跳跃！

### 原文链接及作者信息

- 原文链接：[More Basic Opcodes - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-03/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- hacky: 很 hack 的，常用于描述编程或技术解决方案时，意味着这个解决方案是临时的、不优雅的、或者是以一种不太理想的方式完成的。
- 操作码（Opcode）：介于 0 和某个上限之间的整数，用于表示不同的操作。
- 程序计数器（Program Counter, PC）：用于跟踪正在执行的字节。
- 汇编语言（Assembly Language）：一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- Rust 枚举（Rust Enums）：Rust 语言中的一种类型，用于表示一组相关的值。

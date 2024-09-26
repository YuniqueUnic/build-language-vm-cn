# 05-相等性检查 - 你想构建一个语言虚拟机吗？

- [05-相等性检查 - 你想构建一个语言虚拟机吗？](#05-相等性检查---你想构建一个语言虚拟机吗)
    - [操作码](#操作码)
    - [为什么？](#为什么)
    - [实现](#实现)
        - [存储结果](#存储结果)
        - [相等布尔值](#相等布尔值)
        - [EQ 代码](#eq-代码)
            - [样板 (Boilerplate)](#样板-boilerplate)
        - [JEQ](#jeq)
    - [总结](#总结)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

嘿，你已经走到了这里！恭喜！我希望能说我们快结束了，好给你一些希望，但是，嗯……抱歉。=)

今天，我们将添加一些相等性和比较指令！这些将允许我们测试两个寄存器中的值是否相等、不相等、大于或小于。这些很容易实现，所以应该不会花我们太长时间。

## 操作码

我们将要创建的新 `操作码` 是：

1. EQ（等于）

2. NEQ（不等于）

3. GT（大于）

4. LT（小于）

5. GTQ（大于**或**等于）

6. LTQ（小于**或**等于）

7. JEQ（如果相等则跳转）

JEQ 有点不同，我们将在本文稍后更详细地讨论它。

## 为什么？

这些是条件逻辑的基础。有了这些，我们就可以开始实现 if-then 逻辑、while 循环、for 循环，以及所有那些有趣的东西！

## 实现

假设我们想使用 EQ 操作码。在我们的虚构汇编语言中，它可能看起来像这样：

```assembly
EQ $0 $1
```

我们的虚拟机将检查寄存器 0 和寄存器 1 中的**值**是否相等。但这给我们带来了一个问题……它把结果放在哪里呢？

### 存储结果

我看到有两种方法被使用。一种是我们可以提供一个第三个寄存器来存放结果，所以使用方式将是：

```assembly
EQ $0 $1 $2
```

然后寄存器 2 将有 0（假，或不相等）和 1（真，或相等）。这是 MIPS (`Microprocessor without Interlocked Pipeline Stages`) 的做法。

另一种选择是有一个专门用于存放指令结果的寄存器。你不能 `LOAD` 值到它里面，或者用它做任何事情，只能作为某些指令的运算数，比如 JEQ。

MIPS 通常使用第一种选择，尽管我们的汇编语言在很大程度上基于 MIPS，但我们要在这里稍微偏离一下，使用一个专用寄存器。这纯粹是我的个人偏好；我发现，如果我有一个要跟踪的寄存器，写汇编会更容易一些。

### 相等布尔值

我们需要做的第一件事是向虚拟机添加布尔变量：

```rust
pub struct VM {
    /// 模拟硬件寄存器的数组
    registers: [i32; 32],
    /// 跟踪正在执行的字节的程序计数器
    pc: usize,
    /// 正在运行的程序的字节码
    program: Vec<u8>,
    /// 包含模除操作的余数
    remainder: usize,
    /// 包含最后一次比较操作的结果
    equal_flag: bool,
}
```

然后在我们的 VM 的 `impl` 块中，我们像这样初始化它：

```rust
pub fn new() -> VM {
    VM {
        registers: [0; 32],
        program: vec![],
        pc: 0,
        remainder: 0,
        equal_flag: false,
    }
}
```

### EQ 代码

由于这些操作码的实现，除了 JEQ，几乎相同，我将带你了解如何添加 EQ 代码和 JEQ 代码，其余的留给你。你可以查看[源代码](https://gitlab.com/subnetzero/iridium)以了解其余部分的详细信息。

#### 样板 (Boilerplate)

你现在应该对此有很好的掌握：

1. 在 `instruction.rs` 中的枚举中添加 `EQ`
2. 在 `instruction.rs` 中的 `From<u8>` 实现中添加它
3. 在 `vm.rs` 中实现指令
4. 为它添加一个测试

`EQ` 的实现看起来像：

```rust
Opcode::EQ => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    if register1 == register2 {
        self.equal_flag = true;
    } else {
        self.equal_flag = false;
    }
    self.next_8_bits();
},
```

我们获取两个寄存器的值，并相应地设置虚拟机的 `equal_flag`，然后吃掉接下来的 8 位。

我们的测试看起来像：

```rust
#[test]
fn test_eq_opcode() {
    let mut test_vm = get_test_vm();
    test_vm.registers[0] = 10;
    test_vm.registers[1] = 10;
    test_vm.program = vec![10, 0, 1, 0, 10, 0, 1, 0];
    test_vm.run_once();
    assert_eq!(test_vm.equal_flag, true);
    test_vm.registers[1] = 20;
    test_vm.run_once();
    assert_eq!(test_vm.equal_flag, false);
}
```

| | |
|---|---|
| 重要 | 看看我们如何测试值相等的情况和不相等的情况？总是尽量测试所有可能的路径是好的。|

### JEQ

好的，继续进行本条目中的非比较指令：JEQ，或者如果相等则跳转。它将带有一个寄存器作为参数，如果 `equal_flag` 为真，将跳转到该寄存器中存储的值。如果为假，则不会跳转到它。

这个操作的实现看起来像：

```rust
Opcode::JEQ => {
    let register = self.next_8_bits() as usize;
    let target = self.registers[register];
    if self.equal_flag {
        self.pc = target as usize;
    }
},
```

我们的测试看起来像：

```rust
#[test]
fn test_jeq_opcode() {
    let mut test_vm = get_test_vm();
    test_vm.registers[0] = 7;
    test_vm.equal_flag = true;
    test_vm.program = vec![16, 0, 0, 0, 17, 0, 0, 0, 17, 0, 0, 0];
    test_vm.run_once();
    assert_eq!(test_vm.pc, 7);
}
```

这就是 JEQ 的全部内容！

## 总结

以最小的努力，你还可以实现一个 `JNEQ` 指令，或者如果不相等则跳转。为什么两者都有？方便是主要原因。它将让我们以后编写汇编代码时更容易理解。

拥有可能做不止一件事的大量指令是 RISC(`Reduced Instruction Set Computer`) 处理器和 CISC(`Complex Instruction Set Computer`) 处理器之间的主要区别之一。一个比我们的虚拟机更简单的处理器可能缺少 `MUL` 指令，要求程序员使用 `ADD` 和 `JMP` 指令。

疯狂，我知道。

无论如何，这部分就到这里！接下来，为了改变节奏，我们将开始为我们的虚拟机构建一个 REPL！

### 原文链接及作者信息

- 原文链接：[Equality Checks - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-05/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- MIPS (Microprocessor without Interlocked Pipeline Stages):是一种精简指令集计算机（RISC）架构，由 MIPS 科技公司（现在称为 Wave Computing）开发。MIPS 架构以其简洁、高效的指令集而闻名，广泛应用于各种嵌入式系统、网络设备、游戏控制台和计算机中。
- RISC (Reduced Instruction Set Computer): 是一种简化的指令集，它具有更少的指令，通常 fewer instructions，但是这些指令的功能更丰富。
- CISC (Complex Instruction Set Computer): 是一种复杂的指令集，它具有更多的指令，通常 more instructions，但是这些指令的功能更单一。
- REPL (Read Evaluate Print Loop): 是交互式解释器，用于执行代码，它通常以交互的方式工作，允许用户输入和输出。它是一种简单的、交互式的编程环境，可以让用户输入表达式（代码），然后系统立即计算并返回结果。REPL 环境在编程语言的学习、调试和原型设计中非常有用。
- 操作码（Opcode）：用于表示不同操作的代码。
- 程序计数器（Program Counter, PC）：用于跟踪正在执行的字节。
- 汇编语言（Assembly Language）：一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- Rust 枚举（Rust Enums）：Rust 语言中的一种类型，用于表示一组相关的值。

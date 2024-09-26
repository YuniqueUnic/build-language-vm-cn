# 02-基础操作码 - 你想构建一个语言虚拟机吗？

- [02-基础操作码 - 你想构建一个语言虚拟机吗？](#02-基础操作码---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [什么是操作码 (opcode)？](#什么是操作码-opcode)
        - [枚举](#枚举)
        - [指令 (Instructions)](#指令-instructions)
        - [更多测试](#更多测试)
    - [执行循环](#执行循环)
        - [循环](#循环)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

当我们上次离开我们勇敢的读者（就是你）时，他们已经到了编写操作码的地步。让我们从那里继续。

### 什么是操作码 (opcode)？

一个介于 0 和某个上限之间的整数。因为我们使用 8 位来表示一个操作码，我们可以有 255 个操作码。为了在代码中表示它们，我们将使用枚举，因为 Rust 枚举是最棒的。在你的源代码目录中，创建一个名为 `instruction.rs` 的新文件。

### 枚举

在 `instruction.rs` 中，放入以下代码：

```rust
#[derive(Debug, PartialEq)]
pub enum Opcode {
  HLT,
  IGL
}
```

### 指令 (Instructions)

记得我们的指令是 32 位吗？让我们创建一个结构体来表示一个完整的指令：

```rust
#[derive(Debug, PartialEq)]
pub struct Instruction {
  opcode: Opcode
}
```

我们稍后会添加更多字段，但目前这样就足够了。我们还需要添加一个实现块：

```rust
impl Instruction {
  pub fn new(opcode: Opcode) -> Instruction {
    Instruction {
      opcode: opcode
    }
  }
}
```

### 更多测试

在 `instruction.rs` 中，如果你还没有，添加一个测试模块：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_hlt() {
        let opcode = Opcode::HLT;
        assert_eq!(opcode, Opcode::HLT);
    }

    #[test]
    fn test_create_instruction() {
      let instruction = Instruction::new(Opcode::HLT);
      assert_eq!(instruction.opcode, Opcode::HLT);
    }
}
```

是的，这些测试有点简单，但我习惯于在项目早期尽快编写测试。

| | |
|---|---|
| 重要 | 如果你还没有！不要忘记在 `main.rs` 文件中添加 `pub mod instruction`|

## 执行循环

我们有了一个操作码和一个指令，还有一个虚拟机。我们下一个关键组件是将执行操作码的函数。前往 `vm.rs`，我们将在那里进行一些更改。

首先，我们需要向 VM 结构体添加一个向量来存储我们的程序字节码：

```rust
#[derive(Debug)]
pub struct VM {
    registers: [i32; 32],
    pc: usize,
    program: Vec<u8>
}
```

| | |
|---|---|
| 注 | 不，我们不是存储一个指令向量，而是存储一个字节向量。指令结构体将在我们稍后编写汇编器时有用。|

其次，我们的实现块需要改变：

```rust
impl VM {
    pub fn new() -> VM {
        VM {
            registers: [0; 32],
            program: vec![],
            pc: 0,
        }
    }
}
```

你会注意到我们还添加了一个名为 `pc` 的字段。这是我们的 `程序计数器`，将跟踪正在执行的字节。

### 循环

是时候了！我们可以让我们的虚拟机真正地做些事情了！我的意思是，它将会停止运行，但无论如何，这也算是一个进步！(`I mean, it will stop, but still!`) 让我们为我们的虚拟机实现添加一个功能：

```rust
pub fn run(&mut self) {
    loop {
        // 如果我们的程序计数器超出了程序本身的长度，就出了问题
        if self.pc >= self.program.len() {
            break;
        }
        match self.decode_opcode() {
            Opcode::HLT => {
                println!("遇到 HLT");
                return;
            },
            _ => {
              println!("发现无法识别的操作码！终止！");
              return;
            }
        }
    }
}
```

| | |
|---|---|
| 重要 | 主执行循环通常被认为是语言解释器中最性能关键的部分。这是一个相当简单的实现，没有优化。我们以后会对此进行大量测试和工作。它涉及诸如 CPU 分支预测之类的主题，这些主题值得另写一篇文章。|

上面的代码目前还不能工作。我们把整个程序存储为一个字节向量；我们的虚拟机没有办法知道 HLT 操作码的数字是 0。看到名为 `decode_opcode` 的函数调用了吗？那将是一个 `u8` 并将其转换为操作码。它看起来像这样：

```rust
fn decode_opcode(&mut self) -> Opcode {
    let opcode = Opcode::from(self.program[self.pc]);
    self.pc += 1;
    return opcode;
}
```

把它添加到我们 VM 的实现中。注意我们如何使用 `Opcode::from`？这是一个 Rust `Trait`。我们需要告诉我们的程序如何将一个字节转换为特定的操作码，我们可以通过为我们的枚举实现这个 trait 来做到这一点。在 `instruction.rs` 中放入这个：

```rust
impl From<u8> for Opcode {
    fn from(v: u8) -> Self {
        match v {
            0 => return Opcode::HLT,
            _ => return Opcode::IGL
        }
    }
}
```

我们实际上定义了两个匹配分支：一个用于 HLT，另一个用于所有其他数字。如果虚拟机遇到我们没有计划作为操作码的数字，它将返回 IGL（非法的缩写 (`short for Illegal`)）操作码，虚拟机将出现错误并停止。

最后需要指出的是，在 `decode_opcode` 函数中，我们将程序计数器（`pc`）递增了 1。我们这样做是因为一旦解码了操作码，我们就希望将计数器移动到下一个字节。

在本节结束时，猜猜我们应该添加什么？没错，测试！从上一节开始，我们有一个基本的测试，我将在下面重新粘贴，并添加两个更多：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_vm() {
        let test_vm = VM::new();
        assert_eq!(test_vm.registers[0], 0)
    }

    #[test]
    fn test_opcode_hlt() {
      let mut test_vm = VM::new();
      let test_bytes = vec![0,0,0,0];
      test_vm.program = test_bytes;
      test_vm.run();
      assert_eq!(test_vm.pc, 1);
    }

    #[test]
    fn test_opcode_igl() {
      let mut test_vm = VM::new();
      let test_bytes = vec![200,0,0,0];
      test_vm.program = test_bytes;
      test_vm.run();
      assert_eq!(test_vm.pc, 1);
    }
}
```

对于这个测试，我们可以手动创建一个 4 个字节的向量并运行循环，并检查 pc 是否递增。以后，我们将希望添加一个函数来允许执行一次迭代，以防止失败的测试无限循环。

### 原文链接及作者信息

- 原文链接：[Basic Opcodes - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-02/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 操作码（Opcode）：介于 0 和某个上限之间的整数，用于表示不同的操作。
- 程序计数器（Program Counter, PC）：用于跟踪正在执行的字节。
- 汇编语言（Assembly Language）：一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- Rust 枚举（Rust Enums）：Rust 语言中的一种类型，用于表示一组相关的值。

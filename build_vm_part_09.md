# 09-汇编器 2：巡航控制 - 你想构建一个语言虚拟机吗？

- [09-汇编器 2：巡航控制 - 你想构建一个语言虚拟机吗？](#09-汇编器-2巡航控制---你想构建一个语言虚拟机吗)
    - [Megazord…​激活](#megazord激活)
        - [测试](#测试)
    - [达到程序](#达到程序)
        - [又一个测试](#又一个测试)
    - [获取比特位](#获取比特位)
    - [回到程序](#回到程序)
        - [再来一个测试……](#再来一个测试)
    - [修改 REPL](#修改-repl)
        - [出现了一个野生的错误](#出现了一个野生的错误)
        - [十六进制代码](#十六进制代码)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## Megazord…​激活

我们已经编写了基础的解析器。现在我们可以在抽象阶梯上再上一层，创建一个解析器来组合我们的一些较小解析器。目前，我们可以识别一个操作码、寄存器和整数操作数。我们可以将这些组合成一个 `AssemblerInstruction`。

在 `src/assembler/instruction_parsers.rs` 中，放入以下代码：

```rust
use assembler::Token;
use assembler::opcode_parsers::*;
use assembler::operand_parsers::integer_operand;
use assembler::register_parsers::register;

#[derive(Debug, PartialEq)]
pub struct AssemblerInstruction {
    opcode: Token,
    operand1: Option<Token>,
    operand2: Option<Token>,
    operand3: Option<Token>,
}
```

现在，指令本身的解析器……

```rust
/// 处理以下形式的指令：
/// LOAD $0 #100
named!(pub instruction_one<CompleteStr, AssemblerInstruction>,
    do_parse!(
        o: opcode_load >>
        r: register >>
        i: integer_operand >>
        (
            AssemblerInstruction{
                opcode: o,
                operand1: Some(r),
                operand2: Some(i),
                operand3: None
            }
        )
    )
);
```

看看我们是如何使用我们定义的解析器的？`opcode`、`register` 和 `integer`。它们共同构成了一个 `AssemblerInstruction`。我们把操作数字段留作可选的，以允许更大的灵活性。

| | |
|---|---|
| 注 | 你可能在想解析器名称前面的 `pub`，例如：`pub instruction_one`。这使得由 nom 宏生成的函数变为公共的，以便我们可以从其他模块访问它。我们的 `Program` 解析器将需要从它的模块访问 `instruction_one` 解析器。|

### 测试

现在进行测试……将此代码放在 `src/assembler/instruction_parsers.rs` 文件的底部

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use assembler::opcode::Opcode;

    #[test]
    fn test_parse_instruction_form_one() {
        let result = instruction_one(CompleteStr("load $0 #100\n"));
        assert_eq!(
            result,
            Ok((
                CompleteStr(""),
                AssemblerInstruction {
                    label: None,
                    opcode: Token::Opcode { code: Opcode::LOAD },
                    operand1: Some(Token::Register { reg_num: 0 }),
                    operand2: Some(Token::Number { value: 100 }),
                    operand3: None
                }
            ))
        );
    }
}
```

## 达到程序

现在我们的最终解析器，`Program` 解析器。一个 `Program` 由 `Instructions` 组成。创建 `src/assembler/program_parsers.rs`，并在其中放入以下代码：

```rust
use nom::types::CompleteStr;

use assembler::instruction_parsers::{AssemblerInstruction, instruction_one};

#[derive(Debug, PartialEq)]
pub struct Program {
    instructions: Vec<AssemblerInstruction>
}

named!(pub program<CompleteStr, Program>,
    do_parse!(
        instructions: many1!(instruction_one) >>
        (
            Program {
                instructions: instructions
            }
        )
    )
);
```

我们现在有一个包含汇编指令向量的 `struct`。下一步是让 `AssemblerInstructions` 有能力将它们自己写成 Vec<u8>。然后我们只需要迭代 `instructions` vec 就可以了！

但是首先……

### 又一个测试

```rust
#[test]
fn test_parse_program() {
    let result = program(CompleteStr("load $0 #100\n"));
    assert_eq!(result.is_ok(), true);
    let (leftover, p) = result.unwrap();
    assert_eq!(leftover, CompleteStr(""));
    assert_eq!(
        1,
        p.instructions.len()
    );
    // TODO: 找到一个人体工程学的方法来测试返回的 AssemblerInstruction
}
```

`p` 的 `instructions` 字段（这是一个 `Program` 结构）是私有的。我不确定是让它公开更好，还是制作一个访问器函数，或者怎样。我们稍后再回顾这个问题。

## 获取比特位

我们需要每个 `AssemblerInstruction` 有一个我们可以调用的函数来获取 `Vec<u8>`。让我们前往 `instruction_parser.rs` 并添加一个。

```rust
impl AssemblerInstruction {
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut results = vec![];
        match self.opcode {
            Token::Op { code } => match code {
                _ => {
                    results.push(code as u8);
                }
            },
            _ => {
                println!("Non-opcode found in opcode field");
                std::process::exit(1);
            }
        };

        for operand in vec![&self.operand1, &self.operand2, &self.operand3] {
            match operand {
                Some(t) => AssemblerInstruction::extract_operand(t, &mut results),
                None => {}
            }
        }

        return results;
    }
```

这是在 `src/instruction.rs` 中实现 `impl From<u8> for Opcode {` 的好处。如果你在 `Opcode` 枚举上派生 `Copy` 和 `Clone`，那么我们就可以将任何操作码转换为其整数 `code as u8`。所有这个函数所做的就是将操作码位写入一个向量，然后使用一个辅助函数来提取任何非 None 的操作数字段的操作数。

那个辅助函数也放在 `impl AssemblerInstruction` 中，看起来像：

```rust
fn extract_operand(t: &Token, results: &mut Vec<u8>) {
    match t {
        Token::Register { reg_num } => {
            results.push(*reg_num);
        }
        Token::IntegerOperand { value } => {
            let converted = *value as u16;
            let byte1 = converted;
            let byte2 = converted >> 8;
            results.push(byte2 as u8);
            results.push(byte1 as u8);
        }
        _ => {
            println!("Opcode found in operand field");
            std::process::exit(1);
        }
    };
}
```

我本以为 `borrowck` (Rust 的 borrow 类型检查器) 会斥责，但传递结果向量的效果如我所料。

`extract_operand` 所做的是检查操作数类型，将其转换为字节，然后将它们塞入结果向量。

| | |
|---|---|
| 注 | 你可能想知道为什么我们这样排序它们：</br>`results.push(byte2 as u8);`</br>`results.push(byte1 as u8);`</br>而不是：</br>`results.push(byte1 as u8);`</br>`results.push(byte2 as u8);`</br>这是因为它们需要根据我们的大端/小端规则以正确的顺序排列。|

## 回到程序

让我们回到 `program_parsers.rs` 并添加一个函数来将整个 `AssemblerInstruction` 向量转换为字节：

```rust
impl Program {
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut program = vec![];
        for instruction in &self.instructions {
            program.append(&mut instruction.to_bytes());
        }
        program
    }
}
```

### 再来一个测试……

```rust
#[test]
fn test_program_to_bytes() {
    let result = program(CompleteStr("load $0 #100\n"));
    assert_eq!(result.is_ok(), true);
    let (_, program) = result.unwrap();
    let bytecode = program.to_bytes();
    assert_eq!(bytecode.len(), 4);
    println!("{:?}", bytecode);
}
```

## 修改 REPL

几乎完成了！目前，我们的 REPL 仍然使用十六进制。前往 `src/repl/mod.rs` 在函数 `run` 的捕获所有匹配分支中，放入：

```rust
_ => {
    let parsed_program = program(CompleteStr(buffer));
    if !parsed_program.is_ok() {
        println!("Unable to parse input");
        continue;
    }
    let (_, result) = parsed_program.unwrap();
    let bytecode = result.to_bytes();
    // TODO: 制作一个函数让我们可以将字节添加到虚拟机
    for byte in bytecode {
        self.vm.add_byte(byte);
    }
    self.vm.run_once();
}
```

现在，如果你做 `cargo run` 并输入 `load $0 #100`：

```sh
欢迎使用 Iridium！让我们提高效率！
>>> load $0 #100
>>> .registers
正在列出所有寄存器及其内容：
[
    100,
    0,
    <snip>
]
寄存器列表结束
```

### 出现了一个野生的错误

尝试输入 `LOAD $0 #100`。你应该得到：

```sh
>>> LOAD $0 #100
无法解析输入
>>>
```

我们的汇编器是区分大小写的！我将把它作为一个练习留给读者去解决它。如果你卡住了，你可以在 [GitLab](https://gitlab.com/subnetzero/iridium) 上查看代码。

### 十六进制代码

到这一点，我们可以删除 `parse_hex` 函数，或者我们可以保留它，以防有人觉得在星期五晚上用十六进制编码是件好事。关于如何处理它的一些选项是：

1. REPL 可以尝试两者，并采用不返回 `Error` 的解析器

2. REPL 可以寻找以 `0x` 开头的输入，并使用 `parse_hex` 来解析该输入

3. 我们可以为我们的 REPL 添加一个命令，让它可以在输入模式之间切换。在一个模式中，它接受十六进制。在另一个模式中，汇编代码。

## 结束

耶，我们现在有一个基本的，但功能齐全的汇编器。接下来，我们将教我们的汇编器如何识别更多的操作码和指令形式，以及如何在用户输入错误时为用户提供有用的提示。下次见！

### 原文链接及作者信息

- 原文链接：[Assembler 2: Cruise Control - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-09/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- Megazord…​ACTIVATE!!!: 这句话来源于 90 年代的美国儿童电视节目《恐龙战队》（Mighty Morphin Power Rangers），其中"Megazord"是几个战斗机器人（Zords）组合成的一个巨大的战斗机器人。这句话充满了动作和戏剧性，是一种激发斗志和团队合作的口号。
- borrowck: 是 Rust 编程语言中的一个术语，它是“borrow checker”的缩写，指的是 Rust 的借用检查器。借用检查器是 Rust 安全保证的核心部分，它确保了在编译时对内存安全的严格检查。
- 汇编器（Assembler）：一种程序，可以将汇编语言代码转换成机器代码。
- 解析器（Parser）：一种程序，用于解析和处理输入的代码或指令。
- 操作码（Opcode）：指令中表示操作类型的部分。
- 寄存器（Register）：用于存储数据的硬件元素，常用于编程中的变量存储。
- 整数操作数（Integer Operand）：指令中表示整数数据的部分。
- 程序（Program）：一系列指令的集合，用于执行特定的任务或操作。
- 向量（Vector）：一种数据结构，用于存储一系列元素。
- 十六进制（Hexadecimal）：一种基数为 16 的计数系统，使用数字 0-9 和字母 A-F 表示值。

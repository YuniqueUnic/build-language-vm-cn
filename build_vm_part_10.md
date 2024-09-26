# 10-提升汇编器 - 你想构建一个语言虚拟机吗？

- [10-提升汇编器 - 你想构建一个语言虚拟机吗？](#10-提升汇编器---你想构建一个语言虚拟机吗)
    - [改进汇编器](#改进汇编器)
    - [`From<&str>` 特性](#fromstr-特性)
    - [解析器](#解析器)
        - [测试](#测试)
        - [更新 `instruction.rs`](#更新-instructionrs)
    - [更多指令形式](#更多指令形式)
        - [单个操作码](#单个操作码)
            - [以及它的测试……](#以及它的测试)
        - [使用 `alt!()`](#使用-alt)
    - [其他指令形式](#其他指令形式)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)


## 改进汇编器

我们的汇编器目前只能识别一个操作码 `load`。我们需要教会它识别所有其他的操作码。我们可以通过几种方式来实现这一点：

1. 我们可以为每个操作码编写一个解析器。
2. 我们可以编写一个解析器来识别字母 `a-z`，然后检查它们是否是有效的操作码 (valid Opcode)。

让我们选择方案 #2，因为它将大大减少复制和粘贴 (copy-paste) 的工作。同时，这也给了我们一个在 `instruction.rs` 文件中为 `!==`The `From<&str>` Trait 操作码 实现 `From<CompleteStr<_>>` Trait 的机会！我们在实现了 `From<u8>` 特质的那一块代码下方，加入以下内容：

## `From<&str>` 特性

在 `instruction.rs` 文件中，在我们实现了 `From<u8>` 的代码块下方，添加以下代码：

```rust
impl<'a> From<CompleteStr<'a>> for Opcode {
    fn from(v: CompleteStr<'a>) -> Self {
        match v {
            CompleteStr("load") => Opcode::LOAD,
            CompleteStr("add") => Opcode::ADD,
            CompleteStr("sub") => Opcode::SUB,
            CompleteStr("mul") => Opcode::MUL,
            CompleteStr("div") => Opcode::DIV,
            CompleteStr("hlt") => Opcode::HLT,
            CompleteStr("jmp") => Opcode::JMP,
            CompleteStr("jmpf") => Opcode::JMPF,
            CompleteStr("jmpb") => Opcode::JMPB,
            CompleteStr("eq") => Opcode::EQ,
            CompleteStr("neq") => Opcode::NEQ,
            CompleteStr("gte") => Opcode::GTE,
            CompleteStr("gt") => Opcode::GT,
            CompleteStr("lte") => Opcode::LTE,
            CompleteStr("lt") => Opcode::LT,
            CompleteStr("jmpe") => Opcode::JMPE,
            CompleteStr("nop") => Opcode::NOP,
            _ => Opcode::IGL,
        }
    }
}
```

## 解析器

在 `src/assembler/opcode_parsers.rs` 中，我们有[这个](https://docs.rs/nom/4.0.0/nom/fn.alpha1.html)：

```rust
named!(pub opcode_load<CompleteStr, Token>,
    do_parse!(
        tag!("load") >> (Token::Op{code: Opcode::LOAD})
    )
);
```

现在我们已经为操作码实现了 `From<CompleteStr<_'>>`，前往 `instruction.rs`。`Nom` 有一个非常实用的函数。让我们将我们的操作码解析器更改为：

```rust
named!(pub opcode<CompleteStr, Token>,
    do_parse!(
        opcode: alpha1! >>
        (
            Token::Op{code: Opcode::from(opcode)}
        )
    )
);
```

| | |
|---|---|
| 重要 | 不要忘记在 `instruction.rs` 的顶部添加 `use nom::types::CompleteStr`!|

现在，任何用户输入的非法操作码都将得到一个 `IGL` 操作码。

### 测试

我们需要稍微修改一下 `opcode_parsers.rs` 中的 `test_opcode_load` 来处理我们的新解析器。将其更改为：

```rust
#![allow(unused_imports)]
use super::opcode;
use assembler::Token;
use instruction::Opcode;
use nom::types::CompleteStr;

#[test]
fn test_opcode() {
    let result = opcode(CompleteStr("load"));
    assert_eq!(result.is_ok(), true);
    let (rest, token) = result.unwrap();
    assert_eq!(token, Token::Op { code: Opcode::LOAD });
    assert_eq!(rest, CompleteStr(""));
    let result = opcode(CompleteStr("aold"));
    let (_, token) = result.unwrap();
    assert_eq!(token, Token::Op { code: Opcode::IGL });
}
```

`cargo test` 应该显示所有测试仍然通过。

### 更新 `instruction.rs`

我们需要做的另一个更新是 `test_str_to_opcode` 测试。将其更改为：

```rust
#[test]
fn test_str_to_opcode() {
    let opcode = Opcode::from(CompleteStr("load"));
    assert_eq!(opcode, Opcode::LOAD);
    let opcode = Opcode::from(CompleteStr("illegal"));
    assert_eq!(opcode, Opcode::IGL);
}
```

## 更多指令形式

在 `instruction_parsers.rs` 中，我们编写了一个解析器来处理这种形式的指令：`<opcode> <register> <integer operand>`。然而，指令还可以采取更多的形式，所以让我们编写这些。

首先，将名为 `instruction` 的解析器更改为 `instruction_one` 并删除它的 `pub`。

### 单个操作码

有些指令不接受任何操作数，如 `HLT`。它们的形式是 `<opcode>`。解析器是：

```rust
named!(instruction_one<CompleteStr, AssemblyInstruction>,
    do_parse!(
        o: opcode >>
        opt!(multispace) >>
        (
            AssemblyInstruction{
                opcode: o,
                operand1: None,
                operand2: None,
                operand3: None,
            }
        )
    )
);
```

| | |
|---|---|
| 重要 | 你需要在 `instruction_parsers.rs` 的顶部添加 `use nom::multispace;`。|

#### 以及它的测试……

```rust
#[test]
fn test_parse_instruction_form_two() {
    let result = instruction_two(CompleteStr("hlt\n"));
    assert_eq!(
        result,
        Ok((
            CompleteStr(""),
            AssemblerInstruction {
                opcode: Token::Op { code: Opcode::HLT },
                operand1: None,
                operand2: None,
                operand3: None
            }
        ))
    );
}
```

### 使用 `alt!()`

现在我们拥有了两种可能的指令格式的解析器。但是，我们如何让汇编器尝试每种指令格式，并解析其中有效的那个呢？Nom 提供了一个[便捷的宏](https://docs.rs/nom/4.0.0/nom/macro.alt.html)，名为 `alt`，我们可以给它提供一个解析器列表，如下所示：

```rust
/// 将尝试解析任何一种指令形式
named!(pub instruction<CompleteStr, AssemblerInstruction>,
    do_parse!(
        ins: alt!(
            instruction_one |
            instruction_two
        ) >>
        (
            ins
        )
    )
);
```

看看它如何让我们尝试一个解析器列表？它将返回它找到的第一个有效的解析器。随着我们添加更多的指令形式，我们将在这里添加它们。还要注意，现在这是 `pub` 解析器，`Program` 应该使用它。这意味着你现在需要进入 `program_parsers.rs` 并将所有的 `instruction_one` 引用更改为 `instruction`。

## 其他指令形式

我们还需要一个解析器来处理这种形式：`<opcode> <register> <register> <register>`，用于像 `ADD $0 $1 $2` 这样的指令。

随着我们继续编写应用程序，我们将需要编写更多形式的解析器。我将把最后一种形式留给你来做。如果你遇到困难，你可以在 [GitLab](https://gitlab.com/subnetzero/iridium) 上查看代码。

## 结束

我将在这里结束这部分。在下一部分中，我们将开始讨论内存和字符串。

尽量不要太过兴奋。=)

### 原文链接及作者信息

- 原文链接：[Assembler 3: Assemble Harder - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-10/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 汇编器（Assembler）：一种程序，可以将汇编语言代码转换成机器代码。
- 解析器（Parser）：一种程序，用于解析和处理输入的代码或指令。
- 操作码（Opcode）：指令中表示操作类型的部分。
- 寄存器（Register）：用于存储数据的硬件元素，常用于编程中的变量存储。
- 整数操作数（Integer Operand）：指令中表示整数数据的部分。
- 程序（Program）：一系列指令的集合，用于执行特定的任务或操作。
- 向量（Vector）：一种数据结构，用于存储一系列元素。
- 十六进制（Hexadecimal）：一种基数为 16 的计数系统，使用数字 0-9 和字母 A-F 表示值。

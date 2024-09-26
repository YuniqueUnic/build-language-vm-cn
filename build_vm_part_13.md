# 13-标签 - 你想构建一个语言虚拟机吗？

- [13-标签 - 你想构建一个语言虚拟机吗？](#13-标签---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [新的令牌](#新的令牌)
        - [调整 (Tweak) AssemblyInstruction](#调整-tweak-assemblyinstruction)
        - [指令形式无处不在](#指令形式无处不在)
        - [总结](#总结)
        - [标签是什么？](#标签是什么)
        - [测试](#测试)
    - [稍微离题 (A Slight Digression)](#稍微离题-a-slight-digression)
        - [遍历 (Passes)](#遍历-passes)
        - [使用标签](#使用标签)
        - [存储符号](#存储符号)
    - [然而在此之前…](#然而在此之前)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

嘿，大家好！我知道在上一部分我承诺会讲标签，但在写作过程中，我意识到有一些先决条件我们可以完成。完成它们将使实现`标签`和`指令`变得更加容易。

## 新的令牌

首先，为了处理`标签`和`指令`，我们需要在 `src/assembler/mod.rs` 中添加三种新的令牌类型：

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    Op { code: Opcode },
    Register { reg_num: u8 },
    IntegerOperand { value: i32 },
    LabelDeclaration { name: String },
    LabelUsage { name: String },
    Directive { name: String }
}
```

暂时不用担心这些是什么。

### 调整 (Tweak) AssemblyInstruction

这个很简单！前往 `instruction_parsers.rs` 并为我们的 AssemblerInstruction 添加新字段：

```rust
#[derive(Debug, PartialEq)]
pub struct AssemblerInstruction {
    pub opcode: Option<Token>,
    pub label: Option<Token>,
    pub directive: Option<Token>,
    pub operand1: Option<Token>,
    pub operand2: Option<Token>,
    pub operand3: Option<Token>,
}
```

首先要注意的是，我们将所有内容设为可选。这是因为我们现在可以用`指令`**或**`操作码`开始一个指令。第二点是添加了标签和指令字段。我们稍后会需要这些。

### 指令形式无处不在

目前，我们为每种指令形式编写一个解析器 (`parser`)。一个用于 `<opcode>`，一个用于 `<opcode> <operand>`，依此类推。我们应该对其进行一些优化，将所有这些解析器合并为一个。

首先，我们需要将寄存器作为操作数 (`operands`) 包含在内。打开 `src/assembly/operand_parsers.rs` 并添加以下内容：

```rust
use assembler::register_parsers::register;
```

然后到 `operand` 解析器中，像这样添加它：

```rust
named!(pub operand<CompleteStr, Token>,
    alt!(
        integer_operand |
        register
    )
);
```

接下来，前往 `instruction_parsers.rs` 并添加这个解析器：

```rust
named!(instruction_combined<CompleteStr, AssemblerInstruction>,
    do_parse!(
        l: opt!(label_declaration) >>
        o: opcode >>
        o1: opt!(operand) >>
        o2: opt!(operand) >>
        o3: opt!(operand) >>
        (
            AssemblerInstruction{
                opcode: Some(o),
                label: l,
                directive: None,
                operand1: o1,
                operand2: o2,
                operand3: o3,
            }
        )
    )
);
```

现在，我们需要为指令做同样的事情。打开 `directive_parsers.rs` 并用这些替换里面的所有宏：

```rust
named!(directive_declaration<CompleteStr, Token>,
  do_parse!(
      tag!(".") >>
      name: alpha1 >>
      (
        Token::Directive{name: name.to_string()}
      )
  )
);

named!(directive_combined<CompleteStr, AssemblerInstruction>,
    ws!(
        do_parse!(
            tag!(".") >>
            name: directive_declaration >>
            o1: opt!(operand) >>
            o2: opt!(operand) >>
            o3: opt!(operand) >>
            (
                AssemblerInstruction{
                    opcode: None,
                    directive: Some(name),
                    label: None,
                    operand1: o1,
                    operand2: o2,
                    operand3: o3,
                }
            )
        )
    )
);

/// 将尝试解析任何指令形式
named!(pub directive<CompleteStr, AssemblerInstruction>,
    do_parse!(
        ins: alt!(
            directive_combined
        ) >>
        (
            ins
        )
    )
);

```

这反映了指令解析器的结构。接下来，将 `instruction` 解析器更改为这样：

```rust
/// 将尝试解析任何指令形式
named!(pub instruction<CompleteStr, AssemblerInstruction>,
    do_parse!(
        ins: alt!(
            instruction |
            directive
        ) >>
        (
            ins
        )
    )
);

```

### 总结

所有这些更改的最终结果是，我们现在可以接受更多形式，例如：

`<指令>` `<操作码>` `<指令> <操作数>` `<指令> <操作数> <操作数>` `<指令> <操作数> <操作数> <操作数>` `<操作码> <操作数>` `<操作码> <操作数> <操作数>` `<操作码> <操作数> <操作数> <操作数>`

`<directive> <opcode> <directive> <operand> <directive> <operand> <operand> <directive> <operand> <operand> <operand> <opcode> <operand> <opcode> <operand> <operand> <opcode> <operand> <operand> <operand>`

我们减少了所需的解析器数量。确保 `cargo test` 仍然通过，然后继续……

我应该解释一下标签到底是什么。=)

### 标签是什么？

```txt
在汇编语言中，标签为特定指令提供了一个逻辑名称，你可以在后面引用它。例如：
```

```assembly
test1: LOAD $0 #100
```

然后你可以使用 `@test` 作为某些指令的操作数，例如跳转目标：

```assembly
DJMP @test1
```

这将需要一个新文件。创建 `src/assembler/label_parsers.rs`，并在其中放入这两个解析器：

```rust
use nom::types::CompleteStr;
use nom::{alphanumeric, multispace};

use assembler::Token;

/// 查找用户定义的标签，例如 `label1:`
named!(pub label_declaration<CompleteStr, Token>,
    ws!(
        do_parse!(
            name: alphanumeric >>
            tag!(":") >>
            opt!(multispace) >>
            (
                Token::LabelDeclaration{name: name.to_string()}
            )
        )
    )
);

/// 查找用户定义的标签，例如 `label1:`
named!(pub label_usage<CompleteStr, Token>,
    ws!(
        do_parse!(
            tag!("@") >>
            name: alphanumeric >>
            opt!(multispace) >>
            (
                Token::LabelUsage{name: name.to_string()}
            )
        )
    )
);

```

这些将让我们发现标签的声明（`some_label:`）和它的使用（`@some_label`）。

### 测试

```rust
#[test]
fn test_parse_label_declaration() {
    let result = label_declaration(CompleteStr("test:"));
    assert_eq!(result.is_ok(), true);
    let (_, token) = result.unwrap();
    assert_eq!(token, Token::LabelDeclaration { name: "test".to_string() });
    let result = label_declaration(CompleteStr("test"));
    assert_eq!(result.is_ok(), false);
}

#[test]
fn test_parse_label_usage() {
    let result = label_usage(CompleteStr("@test"));
    assert_eq!(result.is_ok(), true);
    let (_, token) = result.unwrap();
    assert_eq!(token, Token::LabelUsage { name: "test".to_string() });
    let result = label_usage(CompleteStr("test"));
    assert_eq!(result.is_ok(), false);
}

```

## 稍微离题 (A Slight Digression)

在我们继续之前，我们需要更多地讨论汇编器的工作原理。

### 遍历 (Passes)

汇编器以 1 或多个`遍历 (Passes)`操作。也就是说，它们读取编写的代码，做 _某事_，然后重复直到完成。我们的汇编器将是一个 _两遍遍历汇编器_ (_`two-pass`_)。每个遍历阶段需要完成的具体任务并没有严格的规定。一个遍历阶段可能是用来识别所有变量，也可能是进行代码优化。

为何要采用两遍遍历的汇编器呢？嗯，这与 _向前引用问题_ (_`forward reference`_) 有关。

### 使用标签

虽然我们现在能够解析标签，但还不能直接将它们 _用于_ 实际操作。标签并未转化为字节码并输出到我们的字节码文件中。它们的作用是在汇编阶段提供便利。

考虑一下如果我们尝试运行这段代码会发生什么：

```assembly
JMP @target
target: HLT
```

我们在这里试图在使用标签 _之前_ 就将其作为操作数。这有时被称为 _向前引用问题_ (_`forward reference`_)，而我们将通过执行两次遍历来解决这个问题。

### 存储符号

另一个问题是在哪里存储每个符号的值（或着说标签是哪一种类型）？

答案是一种称为 `符号表 (Symbol Table)` 的数据结构。这是一种在汇编器解析代码过程中维护的数据结构，它记录了代码的元信息，如符号对应的具体字节偏移。

我们的 `符号表` 将看起来像这样：

| 符号名称 | 符号类型 | 字节偏移量 |
| --- | --- | --- |
| some_label | 标签 | 12 |

## 然而在此之前…

 在我们深入讨论遍历过程、标签以及符号之前，先将两项新功能加入我们的 REPL：

1. 清除程序向量的命令

2. 从文件读取的能力

第一项功能，我将交由你来实现。而第二项功能，则需要为 `.load_file` 新增一个匹配分支：

```rust
".load_file" => {
    print!("请输入要加载的文件路径：");
    io::stdout().flush().expect("无法刷新标准输出");
    let mut tmp = String::new();
    stdin.read_line(&mut tmp).expect("无法从用户读取行");
    let tmp = tmp.trim();
    let filename = Path::new(&tmp);
    let mut f = File::open(Path::new(&filename)).expect("文件未找到");
    let mut contents = String::new();
    f.read_to_string(&mut contents).expect("从文件读取时出错");
    let program = match program(CompleteStr(&contents)) {
        // Rust 的模式匹配功能强大，甚至支持嵌套使用
        Ok((remainder, program)) => {
            program
        },
        Err(e) => {
            println!("无法解析输入：{:?}", e);
            continue;
        }
    };
    self.vm.program.append(program.to_bytes());
}
```

此匹配分支 (`match arm`) 与之前的基本一致，区别在于它旨在从文件中读取代码，并将其传递给解析器处理。后续，`.load_file` 中的对 `program` 的匹配操作，可以整合至一个通用的功能模块 (`common function`) 中。

## 结束

我认为这篇文章已经足够了。在下一篇文章中，我们将继续开发我们的汇编器并构建一个符号表。如果你需要，代码在 [GitLab](https://gitlab.com/subnetzero/iridium) 上。下次见！

### 原文链接及作者信息

- 原文链接：[Labels - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-13/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 汇编器（Assembler）：一种程序，用于将汇编语言代码转换成机器代码。
- 标签（Label）：在汇编语言中用于标记特定位置的标识符。
- 符号表（Symbol Table）：汇编器在处理代码时维护的数据结构，用于存储有关代码的元数据，如符号的字节偏移量。
- 指令（Directive）：汇编语言中用于控制汇编器行为的特殊指令。
- 操作码（Opcode）：用于表示不同操作的代码。
- 寄存器（Register）：用于存储数据的硬件元素，常用于编程中的变量存储。
- 整数操作数（Integer Operand）：指令中表示整数数据的部分。
- 程序（Program）：一系列指令的集合，用于执行特定的任务或操作。
- 向量（Vector）：一种数据结构，用于存储一系列元素。
- 十六进制（Hexadecimal）：一种基数为 16 的计数系统，使用数字 0-9 和字母 A-F 表示值。

# 08-汇编器：开端 - 你想构建一个语言虚拟机吗？

- [08-汇编器：开端 - 你想构建一个语言虚拟机吗？](#08-汇编器开端---你想构建一个语言虚拟机吗)
    - [指令……汇编](#指令汇编)
    - [词法分析 (Lexing)](#词法分析-lexing)
        - [词素和令牌 (Lexemes and Tokens)](#词素和令牌-lexemes-and-tokens)
    - [语法 (Grammar)](#语法-grammar)
    - [回到词法分析](#回到词法分析)
    - [嗯嗯嗯，美味可口！(Omnomnomnomnom)](#嗯嗯嗯美味可口omnomnomnomnom)
        - [Nom 依赖项](#nom-依赖项)
        - [文件](#文件)
        - [Token 枚举](#token-枚举)
    - [规则：基础规则](#规则基础规则)
        - [规则：操作码](#规则操作码)
            - [测试](#测试)
        - [规则：寄存器](#规则寄存器)
            - [测试 again](#测试-again)
        - [规则：整数操作数](#规则整数操作数)
            - [测试 again again](#测试-again-again)
    - [在 `mod.rs` 中进行总结](#在-modrs-中进行总结)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 指令……汇编

我们可以通过编写所有程序的十六进制代码来折磨自己，如果这是你的兴趣，那么这一节是**技术上**(`technically`) 可选的。

技术上。无论如何，什么是汇编器？它是一个程序，可以将这个：

```assembly
LOAD $1 #10
```

转换成：

```assembly
00 01 00 0A
```

它还有其他几项职责：

1. 处理 `labels`（标签）
2. 计算常量
3. 优化 (`Optimizations`)

## 词法分析 (Lexing)

_词法分析器_（_`Lexer`_）是一个程序，它接收文本流，根据一组规则进行检查，并发出一个 _令牌_ (_`Tokens`_)，并将其发送到输出流。

### 词素和令牌 (Lexemes and Tokens)

_词法分析_ (`Lexing`) 产生 _词素_ (`Lexemes`)，这些是句子中的“意义单位”。在我们的例子 `LOAD $1 #10` 中，词素将是：`LOAD, $, 1, #, 10`。

这些词素与一个 ID 或名称结合成一个 `token`（令牌）。因此，在我们的例子中，我们的令牌是：
`<opcode, 0>, <register, 1>, <number, 10>`

## 语法 (Grammar)

那么，让我们定义一些关于我们的汇编语言的事情：

1. 一个 `Program`（程序）由 `Instructions`（指令）组成

2. 一个 `Instruction`（指令）由以下组成：

   - 一个 `Opcode`（操作码）
   - 一个 `Register`（寄存器）
   - 一个 `IntegerOperand`（整数操作数）
   - 一个 `Newline`（新行）

3. 一个 `Opcode` 由以下组成：

   - 一排中的一个或多个 `Letters`（字母）
   - 一个 `Space`（空格）

4. 一个 `Register` 由以下组成：

   - 符号 `$`
   - 一个 `Number`（数字）
   - 一个 `Space`（空格）

5. 一个 `IntegerOperand` 由以下组成：

   - 符号 `#`
   - 一个 `Number`（数字）

6. 一个 `Number` 由以下组成：

   - 符号 `0-9`

7. 一个 `Newline` 由以下组成：

   - 符号 `\` 后跟符号 `n`

这被称为一个 _*语法*_。它包含了我们语言到目前为止的规则，我们将在本节中扩展它。

| | |
|---|---|
| 重要 | 要进一步了解词法分析和语法，谷歌搜索 `上下文无关语法 (context free grammars)` 和 `巴科斯 - 诺尔范式(backus-naur form)`。|

## 回到词法分析

那么我们如何获得一个词法分析器呢？

两种选择：

1. 我们可以自己编写词法分析器。这并不特别困难，每个人都至少应该做一次。

2. 我们可以使用像 `lex` 这样的工具。

我认真考虑过自己编写一个，但我希望这些教程的重点放在虚拟机上。

有很多关于编写词法分析器的教程，我可能会在以后的某个时候添加一个。

现在，我们将使用 Rust 工具：[Nom](https://github.com/Geal/nom)。这个工具非常容易处理我们的词法分析和解析需求。

## 嗯嗯嗯，美味可口！(Omnomnomnomnom)

要开始吞噬比特 (`gobbling bits`)，我们必须做以下几件事：

1. 将 `nom` 添加为依赖项

2. 将 `nom` crate 添加到 `main.rs`

3. 为汇编器创建一个模块目录

4. 将汇编器模块添加到 `main.rs`

5. 创建一个 `Token` 枚举

6. 为 `nom` 创建规则

### Nom 依赖项

在 Cargo.toml 文件中，将 `nom` 添加为依赖项：

```toml
[dependencies]
nom = "^4.0"
```

### 文件

1. 在 `src` 下创建一个名为 `assembler` 的新目录，并添加一个 `mod.rs` 文件

2. 在 `main.rs` 中，添加 `pub mod assembler;` 到顶部

3. 在 `main.rs` 中，添加 `#[macro_use]` 到顶部

    1. 在 `main.rs` 中，添加 `extern crate nom;` 在下一行

4. 在 `src/assembler` 中创建 `opcode.rs`

### Token 枚举

`nom` 需要 _某种东西_ 来发出，所以我们需要创建一个 Token 枚举。在 `src/assembler/mod.rs` 中，放入：

```rust
use instruction::Opcode;

#[derive(Debug, PartialEq)]
pub enum Token {
    Op{code: Opcode},
}

```

目前，我们只打算教我们的解析器识别一个操作码。每当它找到一个时，它将创建 `Token::Op{code: Opcode}`。

| | |
|---|---|
| 注 | 是的，Token::Op 包含 `code` 这个 `Opcode` 有点奇怪。最初，定义是 `Token::Opcode{code: Opcode}`，但由于 `Opcode` 这个词重复了，并且在两个不同的地方使用了它，这很令人困惑。我希望这样更清楚。|

## 规则：基础规则

好的，是时候编写我们的第一个规则了。我将尽量解释 `nom` 的内容，但要深入了解它，请查看他们的 GitHub。

基本思想是我们将使用 nom 的宏来编写一堆规则，当应用时，可以解析包含有效 Iridium 汇编代码的文件。这些解析器可以相互构建；这允许创建由更简单的规则组成的复杂规则。我们稍后会看到这是多么有用。

### 规则：操作码

在文章前面，我们定义了一个操作码：

```txt
. 一个 `Opcode` 由以下组成：
    * 一排中的一个或多个 `Letters`
    * 一个 `Space`
```

让我们一次处理一行。在 `src/assembler/opcode_parsers.rs` 中：

```rust
use nom::types::CompleteStr;
```

这加载了 `nom` 类型 `CompleteStr`。我们将完整的字符串传递给我们的规则，而不是流式传输数据。

```rust
named!(opcode_load<CompleteStr, Token>,
```

这使用 `nom` crate 中的 `named!` 宏来定义一个名为 `opcode_load` 的函数。它将接受一个 `CompleteStr` 并返回一个 `Token`。

```rust
  do_parse!(
      tag!("load") >> (Token::Op{code: Opcode::LOAD})
  )
);
```

关键部分是 `tag!("load") >> (Token::Op{code: Opcode::LOAD})`。它在它给出的字符串中寻找 "load" 字符串，如果找到，它将返回一个枚举。

| | |
|---|---|
| 注 | `>> (Token::Op{code: Opcode::LOAD})` 部分来自 `do_parse!` 宏。它允许我们链式解析器，并将结果传递给下游的解析器。我们稍后会详细看到这是如何工作的。|

#### 测试

是时候为我们的解析器编写一个测试了！在 `opcode_parsers.rs` 中，放入：

```rust
mod tests {
    use super::*;

    #[test]
    fn test_opcode_load() {
        // 首先测试操作码是否被正确检测和解析
        let result = opcode_load(CompleteStr("load"));
        assert_eq!(result.is_ok(), true);
        let (rest, token) = result.unwrap();
        assert_eq!(token, Token::Op{code: Opcode::LOAD});
        assert_eq!(rest, CompleteStr(""));

        // 测试一个无效的操作码是否被识别
        let result = opcode_load(CompleteStr("aold"));
        assert_eq!(result.is_ok(), false);
    }
}

```

Yay，我们有一个函数可以识别一个操作码了！

| | |
|---|---|
| 重要 | 太棒了，我刚刚掌握了在 asciiDoc 中添加标注的方法！哈哈哈，真是太好了!!！|

### 规则：寄存器

现在来寄存器。首先，让我们在 `src/assembler/mod.rs` 中创建 `Token` 枚举的 `Register` 变体：

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    Op{code: Opcode},
    Register{reg_num: u8}
}
```

接下来，一个寄存器的解析器。我们将在 `src/assembler/register_parsers.rs` 中放入这个。在我们的汇编语言中，它们采用 `$0` 的形式；一个美元符号后面跟着一个数字 `>= 0`。我们的函数看起来像：

```rust
use nom::types::CompleteStr;
use nom::digit;

use assembler::Token;

named!(register <CompleteStr, Token>, <1>
    ws!( <2>
        do_parse!( <3>
            tag!("$") >> <4>
            reg_num: digit >> <5>
            ( <6>
                Token::Register{ <7>
                  reg_num: reg_num.parse::<u8>().unwrap() <8>
                } <9>
            ) <10>
        )
    )
);
```

01. 我们创建了一个名为 `register` 的函数，它接受一个 `CompleteStr` 并返回一个 `CompleteStr` 和 `Token` 或一个 `Error`

02. 我们使用 `ws!` 宏，它会在寄存器的任一侧消耗任何空白。这让我们可以写变体，如 `LOAD $0` 以及 `LOAD $0`

03. 我们使用 `do_parse!` 宏来链式解析器

04. 我们使用 `tag!` 寻找 `$`，将 `tag!` 的结果传递给函数 `digit`，并保存结果在一个名为 `reg_num` 的变量中。nom 提供了 `digit` 函数，它识别一个或多个 0-9 字符

05. 创建具有适当信息的 `Token` 枚举并返回

06. 开始创建 `Token`。我们想要 `Register` 变体

07. 尝试解包并将解析数字的结果存储为 `u8`

08. 关闭 `Token` 结构体

09. 关闭宏将返回的元组结果

#### 测试 again

现在测试它……

```rust
  #[test]
  fn test_parse_register() {
      let result = register(CompleteStr("$0"));
      assert_eq!(result.is_ok(), true);
      let result = register(CompleteStr("0"));
      assert_eq!(result.is_ok(), false);
      let result = register(CompleteStr("$a"));
      assert_eq!(result.is_ok(), false);
  }
```

你可以根据你测试的喜好，添加更多的错误案例，例如 "$"。

### 规则：整数操作数

最后，整数操作数！在 `src/assembler/mod.rs` 中创建 `Token` 枚举的 `IntegerOperand` 变体：

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    Op{code: Opcode},
    Register{reg_num: u8},
    IntegerOperand{value: i32},
}
```

| | |
|---|---|
| 注 | 是的，我们在这里技术上允许用户输入负数，因为我们将其解析为 `i32`。我们的 `LOAD` 指令只能加载 16 位，尽管如此。这是为了未来的扩展。|

接下来，制作文件 `src/assembler/operand_parsers.rs`，在其中我们将放入我们要编写的最后一个解析器：一个能够识别 `IntegerOperand` 的解析器。我们说那些由 `#` 后跟数字组成。在 `operand_parsers.rs` 中，放入：

```rust
use nom::types::CompleteStr;
use nom::digit;

use assembler::Token;

/// 解析器用于整数，我们在汇编语言中用 `#` 前缀：
/// #100
named!(integer_operand<CompleteStr, Token>,
    ws!(
        do_parse!(
            tag!("#") >>
            reg_num: digit >>
            (
                Token::IntegerOperand{value: reg_num.parse::<i32>().unwrap()}
            )
        )
    )
);
```

#### 测试 again again

猜猜这是什么？一个测试！

```rust
#[test]
fn test_parse_integer_operand() {
    // 测试一个有效的整数操作数
    let result = integer_operand(CompleteStr("#10"));
    assert_eq!(result.is_ok(), true);
    let (rest, value) = result.unwrap();
    assert_eq!(rest, CompleteStr(""));
    assert_eq!(value, Token::IntegerOperand{value: 10});

    // 测试一个无效的（缺少 #）
    let result = integer_operand(CompleteStr("10"));
    assert_eq!(result.is_ok(), false);
}

```

## 在 `mod.rs` 中进行总结

现在在 `src/assembler/mod.rs` 中，通过添加：

```rust
pub mod opcode_parsers;
pub mod operand_parsers;
pub mod register_parsers;
```

到顶部来导出我们制作的三个模块。

## 结束

呼，这篇文章比较长，所以我将在这里停止。接下来，我们将讨论如何将这些解析器组合成可以解析整个指令，最终是整个程序的解析器。

### 原文链接及作者信息

- 原文链接：[Assembler: The Beginning - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-08/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 汇编器（Assembler）：一种程序，可以将汇编语言代码转换成机器代码。
- 词法分析器（Lexer）：一种程序，用于将输入的文本分解成一系列的词素或令牌。
- 令牌（Token）：词法分析过程中产生的词素与标识符的组合。
- 操作码（Opcode）：指令中表示操作类型的部分。
- 寄存器（Register）：用于存储数据的硬件元素，常用于编程中的变量存储。
- 整数操作数（Integer Operand）：指令中表示整数数据的部分。
- 语法（Grammar）：用于描述语言结构的规则集合。
- 与上下文无关的语法 ([CFG - Context-free grammar](https://en.wikipedia.org/wiki/Context-free_grammar)): 一种语法，只考虑指令的组合，而不考虑指令之间的顺序。]
- 巴科斯 - 诺尔范式 ([backus-naur form](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)): 一种语法表示方法，用于描述非递归的文法。
- 嗯嗯嗯，美味可口！(Omnomnomnomnom): 这个词组是模仿吃东西时发出的声音，通常用来表达对食物的喜爱或者吃得很香的样子。在编程上下文中，这个词组被用来幽默地表达对工具（在这里是 Nom 这个 Rust 工具）的喜爱和对其功能的肯定。
- RISC (Reduced Instruction Set Computer): 精简指令集计算机，一种计算机架构，特点是指令集简单，指令执行速度快。
- CISC (Complex Instruction Set Computer): 复杂指令集计算机，一种计算机架构，特点是指令集复杂，功能强大。
- REPL (Read Evaluate Print Loop): 交互式解释器，一种编程环境，允许用户输入代码并立即执行和查看结果。

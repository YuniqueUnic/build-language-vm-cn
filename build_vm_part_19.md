# 19-开始 Palladium 的构建 - 你想构建一个语言虚拟机吗？

- [19-开始 Palladium 的构建 - 你想构建一个语言虚拟机吗？](#19-开始-palladium-的构建---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [流程](#流程)
    - [目标](#目标)
    - [解析器](#解析器)
        - [步骤 1：新项目](#步骤-1新项目)
        - [步骤 2：令牌 (Tokens)](#步骤-2令牌-tokens)
        - [步骤 3：操作符解析器](#步骤-3操作符解析器)
        - [步骤 4：操作数解析器](#步骤-4操作数解析器)
    - [语句和表达式](#语句和表达式)
        - [表达式](#表达式)
        - [语句](#语句)
    - [测试](#测试)
    - [步骤 5：表达式解析器](#步骤-5表达式解析器)
    - [步骤 6：程序解析器](#步骤-6程序解析器)
    - [步骤 7：代码生成](#步骤-7代码生成)
        - [访问者模式](#访问者模式)
            - [又有测试](#又有测试)
    - [令牌……编译](#令牌编译)
    - [结束](#结束)
        - [俚语\&典故](#俚语典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

如果你一直在关注 Iridium 的开发，你就会知道它在解析汇编语言时大量使用了 `Nom`。希望你已经喜欢上了它，因为我们也将在这里使用 `Nom`。

在本教程中，我们将开始创建一个名为 `Palladium` 的语言，它将编译为我们一直在使用的汇编代码。

在我们开始之前，请记住……

<img src="https://blog.subnetzero.io/images/i-have-no-idea-what-im-doing.jpg" alt="示意图" width="300" height="auto"/>

我阅读了 [LLVM Kaleidoscope 教程](https://llvm.org/docs/tutorial/LangImpl01.html)，差不多就是这样。如果你看到我做了什么可怕的事情，有建议等，请**告诉我**。

### 流程

这种语言补充了 Iridium 虚拟机，并与之结合。流程图如下所示：

```
+------------------+      +------------------+      +------------------+
| Palladium        |  ->  | Iridium ASM      |  ->  | Iridium VM       |
+------------------+      +------------------+      +------------------+
```

## 目标

在本教程中，我们将从简单的开始。我们的目标是编写一个编译器，将此代码：

```assembly
1 + 2
```

转化为此代码：

```assembly
LOAD $0 #1
LOAD $1 #2
ADD $0 $1 $2
```

## 解析器

下面是我们需要的解析器：

1. 操作符 (Operator) (+,-,/,\*)

2. 整数 (Integer)

3. 表达式 (Expression)

4. 程序 (Program)

让我们开始吧！

### 步骤 1：新项目

1. 运行 `cargo new palladium`

2. 在 `Cargo.toml` 中，将 `nom = "^4.0"` 添加到依赖项

### 步骤 2：令牌 (Tokens)

在 `src/` 下创建一个名为 `tokens.rs` 的文件，并放入以下内容：

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    AdditionOperator,
    SubtractionOperator,
    MultiplicationOperator,
    DivisionOperator,
    Integer{ value: i64 },
    Expression{ left: Box<Token>, op: Box<Token>, right: Box<Token> },
    Program{ expressions: Vec<Token> }
}
```


这些是 Nom 在解析我们的代码时将产生的令牌。(`These are the tokens Nom will be spewing out as it chews through our code.`)

| | |
|---|---|
| 重要 | 注意我们如何必须在表达式和程序中使用某种形式的间接引用；这是因为在编译时我们不知道大小（由于递归）将不得不分配无限量的内存。使用 Box 或 Vec 通过使用已知大小的指针来解决问题。|

### 步骤 3：操作符解析器

现在创建一个名为 `src/operator_parsers.rs` 的文件。这里我们需要 5 个解析器，每个操作符一个，然后使用 `alt!` 来检查它们中的每一个。这是加法操作符解析器：

```rust
named!(addition_operator<CompleteStr, Token>,
    ws!(
        do_parse!(
            tag!("+") >>
            (
                Token::AdditionOperator
            )
        )
    )
);
```

其余的几乎相同 (`identical`)，所以我这里不会展示。检查这四个中的每一个的解析器有点棘手 (`trickier`)：

```rust
named!(operator<CompleteStr, Token>,
    ws!(
        alt!(
            addition_operator |
            subtraction_operator |
            multiplication_operator |
            division_operator
        )
    )
);
```

### 步骤 4：操作数解析器

这与上述类似。目前，我们只有整数作为操作数，但随着我们继续，我们将添加更多。

创建 `src/operand_parsers.rs`，并在其中放置：

```rust
/// 解析整数
named!(integer<CompleteStr, Token>,
    ws!(
        do_parse!(
            sign: opt!(tag!("-")) >>
            reg_num: digit >>
            (
                {
                    let mut tmp = String::from("");
                    if sign.is_some() {
                        tmp.push_str("-");
                    }
                    tmp.push_str(&reg_num.to_string());
                    let converted = tmp.parse::<i64>().unwrap();
                    Token::Integer{ value: converted }
                }
            )
        )
    )
);
```

注意 `sign: opt!(tag!("-"))` 行。这将让我们能够处理 64 位正 (`positive`) 整数或负 (`negative`) 整数。

## 语句和表达式

在继续之前，让我们先了解一下这两个概念以及它们在编程语言上下文中的用途。

### 表达式

表达式 (`Expression`) 是产生新值的东西。这可以是函数和其他值的组合。例如 `5` 是一个值。另一种理解方式是表达式**是**某物。

### 语句

语句 (`Statement`)**不返回**表达式。例如，`x = 5` 是一个语句，因为它不产生任何新值。不过，`x` 和 `5` 都是值。你可以将语句视为**做**某事。

## 测试

哦，对了，操作数解析器的测试：

```rust
mod tests {
    use super::*;
    use tokens::Token;
    use nom::types::CompleteStr;

    #[test]
    fn test_parse_integer() {
        let test_integers = vec!["0", "-1", "1"];
        for o in test_integers {
            let parsed_o = o.parse::<i64>().unwrap();
            let result = integer(CompleteStr(o));
            assert_eq!(result.is_ok(), true);
            let (_, token) = result.unwrap();
            assert_eq!(token, Token::Integer{value: parsed_o});
        }
    }
}
```

我试图更聪明一些，不为 0、-1 和 1 编写三种不同的测试用例。

## 步骤 5：表达式解析器

我们有了操作数和操作符的解析器。由于 `1 + 2` 是操作数和返回值（3）的操作符的组合，这意味着它是一个**表达式**。因此，我们需要一个表达式解析器。创建 `src/expression_parsers.rs` 并在其中放置：

```rust
use nom::types::CompleteStr;

use tokens::Token;
use operand_parsers::operand;
use operator_parsers::operator;

named!(pub expression<CompleteStr, Token>,
    ws!(
        do_parse!(
            left: operand >>
            op: operator >>
            right: operand >>
            (
                Token::Expression{
                    left: Box::new(left),
                    right: Box::new(right),
                    op: Box::new(op)
                }
            )
        )
    )
);
```

## 步骤 6：程序解析器

这是最顶层的解析器，程序！创建 `src/program_parser.rs` 并在其中放置：

```rust
use nom::types::CompleteStr;

use expression_parsers::*;
use tokens::Token;

named!(program<CompleteStr, Token>,
    ws!(
        do_parse!(
            expressions: many1!(expression) >>
            (
                Token::Program {
                    expressions: expressions
                }
            )
        )
    )
);

mod tests {
    use super::*;
    #[test]
    fn test_parse_program() {
        let test_program = CompleteStr("1+2");
        let result = program(test_program);
        assert_eq!(result.is_ok(), true);
    }
}
```

让我们从 `test_parser_program` 打印结果：

```rust
Program {
    expressions: [
        Expression {
            left: Integer {
                value: 1
            },
            op: AdditionOperator,
            right: Integer {
                value: 2
            }
        }
    ]
}
```

注意到树状结构了吗？这是一个 [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)，或抽象语法树 (abstract syntax tree)。我不会在这里详细解释它们，因为有比我能做得更好的解释：

- [维基百科文章](https://en.wikipedia.org/wiki/Abstract_syntax_tree)

- 可视化 AST 的实用工具：[AST Explorer](https://astexplorer.net/), [Tree-sitter](https://tree-sitter.github.io/tree-sitter/using_parsers/),

简而言之 (`A quick summary`)，我们的 Nom 解析器将我们的源文本转换为数据结构。它被称为**抽象**语法树，因为解析器去除了信息，如注释 (`comments`)、空格 (`whitespace`) 等。一个**具体**(`concrete`) 的语法树会保留所有这些。为什么使用 ASTs？这样我们就可以……

## 步骤 7：代码生成

（出于某种原因，这是我编写编程语言时最喜欢的部分）
现在我们有了一个代表我们程序的树，我们希望输出适当的汇编代码，然后 Iridium 虚拟机可以将其编译为可执行字节码。

### 访问者模式

`Visitor Pattern`

一种方法是使用称为 `Visitor` 模式的方法。将其视为编写一个程序，该程序访问我们树中的每个节点，检查它，并输出 IASM 代码。当你需要在不同的对象集合中做某事时，这个模式很有用。

为了做到这一点，让我们使用 Traits！创建 `src/visitor.rs`。这部分有点复杂 (`complicated`)，所以我们将分阶段 (`segments`) 进行。首先，在文件中放入这个：

```rust
use tokens::Token;

pub trait Visitor {
    fn visit_token(&mut self, node: &Token);
}
```

这里我们定义了一个名为 Visitor 的 Trait。有一个函数 `visit_token`，它接受一个对 self 的可变引用和一个对 `Token` 的引用。因为我们在 `tokens.rs` 中使用了枚举，我们不能像这样定义 trait：

```rust
pub trait Visitor {
    fn visit_addition_operator(&mut self, node: &Token::AdditionOperator);
}
```

枚举变体 (`Enum variants`) 不是类型。如果我们将每个枚举变体实现为一个结构体，我们可以使用这种方法。不过，我对 Rust 的枚举有严重的瘾 (`serious addiction`)。

接下来，放入这个：

```rust
pub struct Compiler;

impl Visitor for Compiler {
    fn visit_token(&mut self, node: &Token) {
        match node {
            &Token::AdditionOperator => {},
            &Token::SubtractionOperator => {},
            &Token::MultiplicationOperator => {},
            &Token::DivisionOperator => {},
            &Token::Integer{ value } => {},
            &Token::Expression{ ref left, ref op, ref right } => {},
            &Token::Program{ ref expressions } => {}
        }
    }
}
```

我们声明了一个名为 `Compiler` 的空结构体，然后我们为它实现了 Visitor Trait。

| | |
|---|---|
| 重要 | 匹配部分是我们处理枚举变体不是类型的方式。编译器将访问每个节点，**每个节点都是 Token 类型**，然后它将使用匹配来确定要调用哪些进一步的函数。|

为了说明这一点，我在各种匹配分支中添加了一些调试打印：

```rust
fn visit_token(&mut self, node: &Token) {
    match node {
        &Token::AdditionOperator => {
            println!("Visiting AdditionOperator");
        },
        &Token::SubtractionOperator => {},
        &Token::MultiplicationOperator => {},
        &Token::DivisionOperator => {},
        &Token::Integer{ value } => {
            println!("Visited Integer with value of: {:#?}", value);
        },
        &Token::Expression{ ref left, ref op, ref right } => {
            println!("Visiting an expression");
            self.visit_token(left);
            self.visit_token(op);
            self.visit_token(right);
            println!("Done visiting expression");
        },
        &Token::Program{ ref expressions } => {
            println!("Visiting program");
            for expression in expressions {
                self.visit_token(expression);
            }
            println!("Done visiting program");
        }
    }
}
```

我们稍后会删除这些，但现在它将有助于说明这是如何工作的。

#### 又有测试

这里是 `visitor.rs` 中的一个简单测试：

```rust
mod tests {
    use super::*;
    use nom::types::CompleteStr;
    use program_parsers::program;

    fn generate_test_program() -> Token {
        let source = CompleteStr("1+2");
        let (_, tree) = program(source).unwrap();
        tree
    }

    #[test]
    fn test_visit_addition_token() {
        let mut compiler = Compiler{};
        let test_program = generate_test_program();
        compiler.visit_token(&test_program);
    }
}
```

如果我们运行测试，我们得到：

```sh
$ cargo test test_visit_addition_token -- --nocapture
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running target/debug/deps/palladium-2f91df5ee61c7967

running 1 test
Visiting program
Visiting an expression
Visited Integer with value of: 1
Visiting AdditionOperator
Visited Integer with value of: 2
Done visiting expression
Done visiting program
test visitor::tests::test_visit_addition_token ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 8 filtered out
```

很酷，对吧？我们快到了！

## 令牌……编译

因为我们的虚拟机是基于寄存器的 (`register-based`)，我们必须担心所谓的寄存器分配 (`allocation`)。我们的编译器必须跟踪哪些寄存器正在使用中，哪些没有。以我们示例程序 `1+2` 为例，我们需要三个寄存器。为什么是三个？考虑所需的步骤：

1. 将 `1` 加载到寄存器 `0` (`Load 1 into register 0`)

2. 将 `2` 加载到寄存器 `1` (`Load 2 into a register 1`)

3. 将前两个寄存器相加 (`Add the first two registers`)

4. 将结果存储在第三个寄存器 `2` (`Store the result in a third register 2`)

5. 寄存器 `0` 和 `1` 再次空闲可供使用 (`Registers 0 and 1 are free again for use`)

考虑到这一点，让我们稍微改变一下我们的 Compiler 结构。好吧，非常多：

```rust
#[derive(Default)]
pub struct Compiler{
    free_registers: Vec<u8>,
    used_registers: Vec<u8>,
    assembly: Vec<String>
}

impl Compiler {
    pub fn new() -> Compiler {
        let mut free_registers = vec![];
        for i in 0..31 {
            free_registers.push(i);
        }
        free_registers.reverse();
        Compiler{
            free_registers: free_registers,
            used_registers: vec![],
            assembly: vec![]
        }
    }
}
```

我们添加了三个新属性 (`attributes`)：

- 一个用于存储可供使用的寄存器的向量，
- 一个用于存储当前正在使用的寄存器的向量，
- 以及一个用于保存最终输出的 `Strings` 向量。

| | |
|---|---|
| 注意 | 我们反转了空闲寄存器向量，以便从 0 开始分配。不是因为这很重要，仅仅是出于某种原因让我很愤恨。|

准备好 `visit_token` 函数的最终版本了吗？我们开始吧……

```rust
impl Visitor for Compiler {
    fn visit_token(&mut self, node: &Token) {
        match node {
            &Token::AdditionOperator => {
                // TODO: 需要清理这个。移除 unwraps。
                let result_register = self.free_registers.pop().unwrap();
                let left_register = self.used_registers.pop().unwrap();
                let right_register = self.used_registers.pop().unwrap();
                let line = format!("ADD ${} ${} ${}", left_register, right_register, result_register);
                self.assembly.push(line);
                self.free_registers.push(left_register);
                self.free_registers.push(right_register);

            },
            &Token::SubtractionOperator => {},
            &Token::MultiplicationOperator => {},
            &Token::DivisionOperator => {},
            &Token::Integer{ value } => {
                let next_register = self.free_registers.pop().unwrap();
                let line = format!("LOAD ${} #{}", next_register, value);
                self.used_registers.push(next_register);
                self.assembly.push(line);
            },
            &Token::Expression{ ref left, ref op, ref right } => {
                self.visit_token(left);
                self.visit_token(right);
                self.visit_token(op);
            },
            &Token::Program{ ref expressions } => {
                for expression in expressions {
                    self.visit_token(expression);
                }
            }
        }
    }
}
```

> 1. A Program consists of Expressions, so we iterate over each Expression.
> 2. For each Expression, visit the left, right and op side.
> 3. When we visit an Integer, pop a free register off to store it
> 4. When we visit an AdditionOperator, pop a free register to store the result
> 5. Pop the two most recently added used registers
> 6. Generate the IASM
> 7. Add the registers previously used to store the left and right values to the free list


1. 一个 `Program` 由 `Expressions` 组成，所以我们遍历每个 `Expression`。

2. 对于每个 `Expression`，访问 `left`、`right` 和 `op` 侧。

3. 当我们访问一个整数时，弹出一个空闲寄存器来存储它

4. 当我们访问一个 AdditionOperator 时，弹出一个空闲寄存器来存储结果

5. 弹出最近添加的两个用于存储左和右值的已使用寄存器

6. 生成 IASM

7. 将之前用于存储左和右值的寄存器添加到空闲列表中

| | |
|---|---|
| 重要 | 这对于更复杂的表达式不起作用，但我们将在后面的教程中处理这个问题。|

现在如果我们运行测试，输出给我们：

```sh
[
    "LOAD $0 #1",
    "LOAD $1 #2",
    "ADD $1 $0 $2"
]
```

甚至更酷，对吧？

让我们启动 Iridium REPL 并输入这三行：

```sh
$ iridium
Welcome to Iridium! Let's be productive!
>>> load $0 #1
>>> load $1 #2
>>> add $1 $0 $2
>>> .registers
Listing registers and all contents:
[
    1,
    2,
    3,
    0,
    0,
    <snip>
]
End of Register Listing
```

成功！

## 结束

呼，那是一篇很长的帖子。我们可以通过向 `Compiler` 结构添加一个汇编器来使其无缝，我们将在以后的教程中进行。我现在完成了。=) Palladium 在 [GitLab](https://gitlab.com/subnetzero/palladium) 上

我们下次教程见！

原文出处：[Benchmarks - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-19/)
作者名：Fletcher

### 俚语&典故

1. `A Faustian Bargain`：浮士德式的交易 (A Faustian Bargain)，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

2. `Zergling`: 是来自于游戏《星际争霸》（StarCraft）中的一个单位，属于 Zerg 种族的基本战斗单位。在这里，作者可能是以幽默的方式，将线程比作是快速、大量生产的小型战斗单位，以此来形象地描述线程的创建和管理工作。在翻译时，可以保留原文`zergling`，因为它已经成为了游戏文化中的一个专有名词，或者可以解释性地翻译为`快速生成的战斗单位`或`轻量级线程`。

### 专有名词及注释

1. 抽象语法树 (abstract syntax tree): 是一种用于描述程序结构的数据结构，它将源代码中的每个元素都表示为一个节点，每个节点都包含一个表示其含义的符号，以及指向其子节点的指针。
2. 访问者模式 (Visitor Pattern): 是一种设计模式，它允许我们通过定义一个访问者接口，来分离对象和它所支持的访问操作。这样，我们可以在不改变对象结构的情况下，为对象添加新的操作。
3. IASM (Interactive Abstract Syntax Machine):是一种用于分析程序的控制流和数据流的高效工具。它由一系列的转换器组成，可以将程序转换成不同的表示形式，如控制流图（CFG）、抽象语法树（AST）、中间代码等。IASM 可以帮助程序员更好地理解程序的结构和行为，同时也可以用于程序优化、错误检测和代码生成等任务。IASM 通常包含以下几个部分：
    1. 解析器（Parser）：将源代码转换成 AST。
    2. 转换器（Transformer）：将 AST 转换成其他表示形式，如中间代码。
    3. 分析器（Analyzer）：分析程序的控制流和数据流。
    4. 优化器（Optimizer）：优化程序的性能。
    5. 代码生成器（Code Generator）：将优化后的程序转换成目标代码。

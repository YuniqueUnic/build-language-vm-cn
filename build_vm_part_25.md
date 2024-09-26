# 25-扩展 Palladium：第一部分 - 你想构建一个语言虚拟机吗？

- [25-扩展 Palladium：第一部分 - 你想构建一个语言虚拟机吗？](#25-扩展-palladium第一部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [目标](#目标)
    - [解析器](#解析器)
        - [新的 Token](#新的-token)
        - [因子](#因子)
        - [表达式解析器](#表达式解析器)
        - [术语解析器](#术语解析器)
        - [示例表达式](#示例表达式)
        - [测试](#测试)
            - [术语测试](#术语测试)
            - [因子测试](#因子测试)
            - [表达式测试](#表达式测试)
            - [程序测试](#程序测试)
            - [汇编输出 (Assembly Output)](#汇编输出-assembly-output)
    - [结束](#结束)
        - [专有名词及注释](#专有名词及注释)

## 引言

大家好！在本教程中，我们将转换轨道，对 Palladium 进行一些工作。目前，它只能处理简单的算术表达式，比如 `2+1`，仅此而已。让我们看看是否能够让它处理更复杂的东西。

### 目标

1. 处理括号表达式（例如，`(4-3)*2`）
    1. `Handle parenthetical expressions (e.g., (4-3)*2))`

2. 教会我们的解析器处理浮点数
    1. `Teach our parser to handle floating point numbers`

## 解析器

让我们快速回顾一下：

1. 一个 `Program` 由 `Expressions` 组成

2. 一个 `Expression` 返回一个值

3. 一个 `Operand` 是 `Operator` 左右两边的部分，例如在 `2+1` 中，操作数是 2 和 1

4. 一个 `Operator` 是 `+,-,*,/`

5. Nom 将所有这些解析为 `Tokens`（在 `src/tokens.rs` 中找到）

我们将稍微改变我们的定义：

1. 一个 `Expression` 由 `Terms` 组成

2. 一个 `Term` 由 `Factors` 组成

3. 一个 `Term` 可以被加或减

4. 一个 `Factor` 可以被乘或除

5. 一个 `Factor` 可以被 `(` 和 `)` 包围

| | |
|---|---|
| 重要 | 是时候进行一些数学讨论了！别担心，这并不复杂。=)</br>除非你使用过 Lisp 风格的语言（例如 Clojure）或者那些老式 HP 计算器，你很可能习惯于看到所有数学表达式都以以下形式书写：</br></br>1. 3+1</br>   </br>2. 2 \* (1+1)</br></br>看到操作符在中间了吗？这被称为 **中缀表示法** (`infix notation`)。这不是唯一的表达式书写方式。如果我们把 `3+1` 写成 `3 1 +` 呢？那被称为 **后缀表示法** (`postfix notation`)。如果我们把它写成 `+ 3 1` 呢？那被称为 **前缀表示法** (`prefix notation`)。前缀和后缀表示法实际上更容易解析，但它提高了你语言的入门门槛，因为人们倾向于不喜欢使用非中缀表示法。|

### 新的 Token

前往 `src/tokens.rs`。我们需要改变一些事情：

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum Token {
    AdditionOperator,
    SubtractionOperator,
    MultiplicationOperator,
    DivisionOperator,
    Integer{ value: i64 },
    Float{ value: f64},
    Factor{ value: Box<Token> },
    Term{ left: Box<Token>, right: Vec<(Token, Token)> },
    Expression{ left: Box<Token>, right: Vec<(Token, Token)> },
    Program{ expressions: Vec<Token> }
}
```

我们为 `Float`、`Factor` 和 `Term` 添加了 Token。我们还从 `Expression` 中移除了 `op` 字段，并替换为元组的 `Vec`。我们马上就会看到为什么。

我们还需要在 `visitor.rs` 中为新 Token 制作一些匹配分支：

```rust
&Token::Float{ value } => {},
&Token::Factor{ ref value } => {},
&Token::Term{ ref left, ref right } => {},
&Token::Expression{ ref left, ref right } => {},
&Token::Program{ ref expressions } => {},
```

### 因子

现在让我们教解析器什么是因子。创建 `src/factor_parsers.rs`，并在其中放置：

```rust
use nom::*;
use nom::types::CompleteStr;

use tokens::Token;
use expression_parsers::expression;

named!(float64<CompleteStr, Token>,
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

解析器执行以下操作：

> 1. Consume any whitespace
> 2. Check if there is a -. Because we use opt!, it is optional. This is to handle negative floats.
> 3. Consume all digits until it encounters .
> 4. Consume all digits after the .
> 5. Construct the final floating point string representation
> 6. Convert it to a f64
> 7. Return a Factor token that contains a Float token

1. 消耗任何空白
2. 检查是否有 `-`。因为我们使用 `opt!`，它是可选的。这是为了处理负浮点数。
3. 消耗 `.` 之前的所有数字
4. 消耗 `.` 之后的所有数字
5. 构造最终的浮点字符串表示
6. 将其转换为 `f64`
7. 返回包含 `Float` Token 的 `Factor` Token

一些这个解析器旨 (`meant`) 在检测的浮点数示例：

1. `1.0`
2. `-1.0`
3. `323.8`
4. `1.453`

它几乎与我们的整数解析器相同。既然整数也是 `Factor`，让我们将整数解析器移过来：

```rust
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

无需更改，只是更改了位置。

最后，是 `Factor` 解析器！它看起来像这样：

```rust
named!(pub factor<CompleteStr, Token>,
    ws!(
        do_parse!(
            f: alt!(
                integer |
                float64 |
                ws!(delimited!( tag_s!("("), expression, tag_s!(")") ))
            ) >>
            (
                {
                    Token::Factor{value: Box::new(f)}
                }
            )
        )
    )
);
```

这里是它的工作原理：

> 1. Try the integer parser that we just moved into the same file
> 2. Try the float64 parser we just made
> 3. Look for an Expression surrounded by ( and )

1. 尝试我们刚刚移到同一个文件中的 `integer` 解析器

2. 尝试我们刚刚制作的 `float64` 解析器

3. **查找被 `(` 和 `)` 包围的 `Expression`**

最后一个非常重要，因为它使我们能够处理带有嵌套括号 (`parentheses`) 的表达式。

### 表达式解析器

打开 `src/expression_parsers.rs`。我们需要稍微改变一下这个解析器，以便它能够使用我们的新 `Factor` 解析器。新的 `Expression` 解析器看起来像这样：

```rust
use nom::types::CompleteStr;

use tokens::Token;
use term_parsers::term;
use operator_parsers::*;

named!(pub expression<CompleteStr,Token>,
    do_parse!(
        left: term >>
        right: many0!(
            tuple!(
                alt!(
                    addition_operator |
                    subtraction_operator
                ),
                term
            )
        ) >>
        (
            {
                Token::Expression{left: Box::new(left), right: right}
            }
        )
    )
);
```

| | |
|---|---|
| 重要 | 注意 `use` 更改。我们正在导入 `Term` 解析器，我们还没有编写。别担心！我们会到那里的！|

这个解析器现在做的是：

> 1. Look for a Term
> 2. Looks for zero or characters that match the following pattern: + or - then a Term
> 3. Because we use the tuple! macro, we will get back a (Token, Token) result, which is why we had to change our Token definitions a bit.
> 4. Construct an Expression and return it

1. 查找一个 `Term`

2. 查找零个或多个字符，这些字符匹配以下模式：`+` **或** `-` 然后是一个 `Term`

3. 因为我们使用了 `tuple!` 宏，我们将得到一个 `(Token, Token)` 结果，这就是为什么我们不得不稍微改变我们的 `Token` 定义。

4. 构造一个 `Expression` 并返回它

### 术语解析器

我们的最后一个新解析器是 `Term` 解析器。创建 `src/term_parsers.rs`，并在其中放置：

```rust
use nom::types::CompleteStr;

use tokens::Token;
use factor_parsers::factor;
use operator_parsers::{multiplication_operator, division_operator};

named!(pub term<CompleteStr,Token>,
    do_parse!(
        left: factor >>
        right: many0!(
            tuple!(
                alt!(
                    multiplication_operator |
                    division_operator
                ),
                factor
            )
        ) >>
        (
            {
                Token::Term{left: Box::new(left), right: right}
            }
        )
    )
);
```

它几乎与 `Factor` 解析器做相同的事情：

> 1. Look for a Factor
> 2. Looks for zero or characters that match the following pattern: *or / then a Factor
> 3. Because we use the tuple! macro, we will get back a (Token, Token) result, which is why we had to change our Token definitions a bit.
> 4. Construct a Term and return it

1. 查找一个 `Factor`

2. 查找零个或多个字符，这些字符匹配以下模式：`` **\*or** `/` 然后是一个 `Factor`

3. 因为我们使用了 `tuple!` 宏，我们将得到一个 `(Token, Token)` 结果，这就是为什么我们不得不稍微改变我们的 `Token` 定义。

4. 构造一个 `Term` 并返回它

| | |
|---|---|
| 注意 | 注意 `Term` 查找 `*` 和 `/`，而 `Factor` 查找 `+` 和 `-`。这就是我们如何建立操作符优先级。|

### 示例表达式

让我们看看 Nom 将 `3*4` 转换为什么：

```rust
(
    CompleteStr(
        ""
    ),
    Term {
        left: Factor {
            value: Integer {
                value: 3
            }
        },
        right: [
            (
                MultiplicationOperator,
                Factor {
                    value: Integer {
                        value: 4
                    }
                }
            )
        ]
    }
)
```

在左手边，我们有一个 `Term`，它有一个 `Factor`，它有一个 `Integer`。在右边，我们有一个包含 `MultiplicationOperator` 和包含 `Integer` 的 `Factor` 的 `Vec`。

更复杂的表达式的 ASTs 可能会变得更大。=)

### 测试

现在让我们写一些测试！

#### 术语测试

在 `term_parsers.rs` 中放置：

```rust
mod tests {
    use super::*;

    #[test]
    fn test_parse_term() {
        let result = term(CompleteStr("3*4"));
        assert_eq!(result.is_ok(), true);
    }

    #[test]
    fn test_parse_nested_term() {
        let result = term(CompleteStr("(3*4)*2"));
        assert_eq!(result.is_ok(), true);
    }

    #[test]
    fn test_parse_really_nested_term() {
        let result = term(CompleteStr("((3*4)*2)"));
        assert_eq!(result.is_ok(), true);
    }
}
```

#### 因子测试

这些放在 `factor_parsers.rs` 中：

```rust
mod tests {
    use super::*;
    #[test]
    fn test_factor() {
        let test_program = CompleteStr("(1+2)");
        let result = factor(test_program);
        assert_eq!(result.is_ok(), true);
        let (_, tree) = result.unwrap();
        println!("{:#?}", tree);
    }

    #[test]
    fn test_parse_floats() {
        let test_floats = vec!["100.4", "1.02", "-1.02"];
        for o in test_floats {
            let parsed_o = o.parse::<f64>().unwrap();
            let result = float64(CompleteStr(o));
            assert_eq!(result.is_ok(), true);
        }
    }

    #[test]
    fn test_parse_integer() {
        let test_integers = vec!["0", "-1", "1"];
        for o in test_integers {
            let parsed_o = o.parse::<i64>().unwrap();
            let result = integer(CompleteStr(o));
            assert_eq!(result.is_ok(), true);
        }
    }
}
```

#### 表达式测试

如果我们想解析包含所有操作符的东西，我们需要使用 Expression。让我们在 `expression_parsers.rs` 中添加一些测试：

```rust
#[test]
fn test_parse_expression() {
    let result = expression(CompleteStr("3*4"));
    assert_eq!(result.is_ok(), true);
}

#[test]
fn test_parse_nested_expression() {
    let result = expression(CompleteStr("(3*4)+1"));
    assert_eq!(result.is_ok(), true);
}
```

#### 程序测试

最后，让我们在 `program_parsers.rs` 中做一个花哨的测试：

```rust
#[test]
fn test_parse_program() {
    let test_program = CompleteStr("1+2");
    let result = program(test_program);
    assert_eq!(result.is_ok(), true);
}
```

如果你运行测试，你可能会遇到类似的失败：

```sh
thread 'visitor::tests::test_visit_addition_token' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:345:21
```

这意味着在我们的加法函数中：

```rust
&Token::AdditionOperator => {
    // TODO: Need to clean this up. Remove the unwraps.
    let result_register = self.free_registers.pop().unwrap();
    let left_register = self.used_registers.pop().unwrap();
    let right_register = self.used_registers.pop().unwrap();
    let line = format!("ADD ${} ${} ${}", left_register, right_register, result_register);
    self.assembly.push(line);
    self.free_registers.push(left_register);
    self.free_registers.push(right_register);
},
```

其中一个已使用的寄存器还没有放入使用栈中。记住我们是如何开始返回元组的吗？正在发生的是，在 `Expression` 匹配分支中：

```rust
&Token::Expression{ ref left, ref right } => {
    self.visit_token(left);
    for expression in right {
      self.visit_token(&expression.1);
      self.visit_token(&expression.0);
    }
},
```

当它评估 (`evaluates`) 我们的树的这一部分时：

```rust
right: [
    (
        AdditionOperator,
        Term {
            left: Factor {
                value: Integer {
                    value: 2
                }
            },
            right: []
        }
    )
]
```

它将在访问 `Integer` 值 **之前** 访问加法运算符。这意味着值 2 还没有放入寄存器中。我们希望 `Expression` 函数以相反的顺序访问它们。所以改变代码：

```rust
&Token::Expression{ ref left, ref right } => {
    self.visit_token(left);
    for expression in right {
      self.visit_token(&expression.0);
      self.visit_token(&expression.1);
    }
},
```

```rust
&Token::Expression{ ref left, ref right } => {
    self.visit_token(left);
    for expression in right {
      self.visit_token(&expression.1);
      self.visit_token(&expression.0);
    }
},
```

你还需要对 `Term` 分支做同样的更改。

如果我们现在运行它..

```sh
thread 'visitor::tests::test_nested_operators' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:345:21
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

GRRRR!

这个问题花了我一些时间调试，我不得不写一些调试函数来打印使用中的寄存器。我还添加了这个测试到 `visitor.rs`。

```rust
#[test]
fn test_nested_operators() {
    let mut compiler = Compiler::new();
    let test_program = generate_test_program("(4*3)-1");
    compiler.visit_token(&test_program);
    let bytecode = compiler.compile();
}
```

程序解析得很好，直到 `-`；乘法 (`multiplication`) 部分没问题。通过打印出在它击中减法分支时使用中的寄存器：

```sh
--------------------
|  Used Registers  |
--------------------
0
--------------------
|  Free Registers  |
--------------------
30
<snip>
1

```

我们应该有两个使用中的寄存器：一个用于乘法指令的结果，一个用于 `1` 的加载指令。显然 (`Clearly`) 那个寄存器从未被放入使用寄存器列表中。如果我们看一下处理 `SUB` 操作码的分支：

```rust
self.print_used_registers();
self.print_free_registers();
let result_register = self.free_registers.pop().unwrap();
let left_register = self.used_registers.pop().unwrap();
let right_register = self.used_registers.pop().unwrap();
let line = format!("SUB ${} ${} ${}", left_register, right_register, result_register);
self.assembly.push(line);
self.free_registers.push(left_register);
self.free_registers.push(right_register);
```

看到问题了吗？我们从没把`result_register` 推送到使用的寄存器栈中。我们可以像这样来修复它：

```rust
self.print_used_registers();
self.print_free_registers();
let result_register = self.free_registers.pop().unwrap();
let left_register = self.used_registers.pop().unwrap();
let right_register = self.used_registers.pop().unwrap();
let line = format!("SUB ${} ${} ${}", left_register, right_register, result_register);
self.assembly.push(line);
self.used_registers.push(result_register); // <-- fixed
self.free_registers.push(left_register);
self.free_registers.push(right_register);
```

这个问题存在于所有的 operator 的分支种，所以我们需要通过同样的方式来修复所有的分支。

现在再让我们来运行他们，他们应该是能够通过的。

```sh
running 18 tests
test operator_parsers::tests::test_parse_division_operator ... ok
test operator_parsers::tests::test_parse_addition_operator ... ok
test operator_parsers::tests::test_parse_multiplication_operator ... ok
test factor_parsers::tests::test_parse_integer ... ok
test factor_parsers::tests::test_parse_floats ... ok
test expression_parsers::tests::test_parse_expression ... ok
test expression_parsers::tests::test_parse_nested_expression ... ok
test operator_parsers::tests::test_parse_operator ... ok
test operator_parsers::tests::test_parse_subtraction_operator ... ok
test program_parsers::tests::test_parse_program ... ok
test factor_parsers::tests::test_factor ... ok
test term_parsers::tests::test_parse_nested_term ... ok
test term_parsers::tests::test_parse_term ... ok
test visitor::tests::test_visit_addition_token ... ok
test visitor::tests::test_visit_subtraction_token ... ok
test term_parsers::tests::test_parse_really_nested_term ... ok
test visitor::tests::test_nested_operators ... ok
test visitor::tests::test_visit_multiplication_token ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

好耶!(YAY!)

|||
|---|---|
|提示 | 我以及添加了部分测试。你可以通过查看源码来查看更多细节|

#### 汇编输出 (Assembly Output)

让我们快速地添加一个更改，以便打印出 `test_nested_operators` 测试的汇编输出。

```sh
".data"
".code"
"LOAD $0 #4"
"LOAD $1 #3"
"MUL $1 $0 $2"
"LOAD $0 #1"
"SUB $0 $2 $1"
```

看起来是对的！如果我们一步步来：

> 1. Load 4 into R0
> 2. Load 3 into R1
> 3. Multiply R0 and R1 and store result in R2 (which has the value of 12)
> 4. Load 1 into R0
> 5. Subtract R0 from R2 and store result in R3 (which should have 11 at the end)
> 6. Crap, that’s -11

1. 将 4 载入 R0
2. 将 3 载入 R1
3. 将 R0 和 R1 相乘并将结果存储在 R2 中（R2 的值为 12）
4. 将 1 载入 R0
5. 从 R2 中减去 R0 并将结果存储在 R3 中（最终应该是 11）
6. 糟糕，结果是 -11

如果我们通过 Iridium REPL 运行生成的代码，我们会看到：

```sh
Welcome to Iridium! Let's be productive!
>>> LOAD $0 #4
    LOAD $1 #3
>>> MUL $1 $0 $2
>>> LOAD $0 #1
>>> SUB $0 $2 $1
>>> !registers
Listing registers and all contents:

>>> [
    1,
    -11,
    12,
    0,
    0,
    // <snip>
]
```

该死！在这种情况下，问题是出在 Iridium 上。它处理减法 (`subtraction`) 的代码如下：

```rust
Opcode::SUB => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    self.registers[self.next_8_bits() as usize] = register1 - register2;
}
```

以及我们输出代码的方式，我们需要要么改变 Iridium 虚拟机来做：

```rust
Opcode::SUB => {
    let register1 = self.registers[self.next_8_bits() as usize];
    let register2 = self.registers[self.next_8_bits() as usize];
    self.registers[self.next_8_bits() as usize] = register2 - register1;
}
```

或者我们可以改变我们的解析器以相反的顺序输出减法指令。现在，我们将在解析器中这样做，因为这是一个 Palladium 教程。=)

在 `visitor.rs` 中，我们可以改变：

```rust
&Token::SubtractionOperator => {
    // TODO: Need to clean this up. Remove the unwraps.
    let result_register = self.free_registers.pop().unwrap();
    let left_register = self.used_registers.pop().unwrap();
    let right_register = self.used_registers.pop().unwrap();
    let line = format!("SUB ${} ${} ${}", left_register, right_register, result_register);
    self.assembly.push(line);
    self.used_registers.push(result_register);
    self.free_registers.push(left_register);
    self.free_registers.push(right_register);
},
```

变为：

```rust
&Token::SubtractionOperator => {
    // TODO: Need to clean this up. Remove the unwraps.
    let result_register = self.free_registers.pop().unwrap();
    let left_register = self.used_registers.pop().unwrap();
    let right_register = self.used_registers.pop().unwrap();
    let line = format!("SUB ${} ${} ${}", right_register, left_register, result_register); // reverse the right and left
    self.assembly.push(line);
    self.used_registers.push(result_register);
    self.free_registers.push(left_register);
    self.free_registers.push(right_register);
},
```

现在我们的输出 (Palladium) 是：

```sh
".data"
".code"
"LOAD $0 #4"
"LOAD $1 #3"
"MUL $1 $0 $2"
"LOAD $0 #1"
"SUB $2 $0 $1" # 这里是 2 2 0 1, 在上面调换了 left 和 right
```

通过 Iridium REPL 运行它：

```sh
Welcome to Iridium! Let's be productive!
>>> LOAD $0 #4
    LOAD $1 #3
>>> MUL $1 $0 $2
>>> LOAD $0 #1
>>> SUB $0 $2 $1 # 这里是 2 0 2 1, 实际上, 如果是通过调换 left, right 的方式来解决 sub 的问题的话, 这里应该输入 2 2 0 1  
>>> !registers
Listing registers and all contents:
>>> [
    1,
    11,
    12,
    0,
    //    <snip>
]
```

好的！这次真的成功了！

## 结束

这部分到此为止！下次见！
(`That’s it for this installment! See everyone next time!`)

原文出处：[Extending Palladium: Part 1 - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-25/)
作者名：Fletcher

### 专有名词及注释

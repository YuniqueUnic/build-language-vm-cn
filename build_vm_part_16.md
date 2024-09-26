# 16-字符串常量和更多 - 你想构建一个语言虚拟机吗？

- [16-字符串常量和更多 - 你想构建一个语言虚拟机吗？](#16-字符串常量和更多---你想构建一个语言虚拟机吗)
    - [即将完成 (Home Stretch)](#即将完成-home-stretch)
        - [快速更改](#快速更改)
        - [字符串常量](#字符串常量)
        - [只读部分](#只读部分)
    - [更多解析器](#更多解析器)
        - [以及一个测试](#以及一个测试)
        - [指令](#指令)
    - [终于](#终于)
    - [汇编器](#汇编器)
    - [两遍遍历汇编器](#两遍遍历汇编器)
        - [impl 块](#impl-块)
        - [assemble()](#assemble)
        - [process\_first\_phase()](#process_first_phase)
            - [process\_label\_directive()](#process_label_directive)
        - [process\_second\_phase()](#process_second_phase)
            - [process\_directive()](#process_directive)
            - [process\_section\_header()](#process_section_header)
            - [handle\_asciiz()](#handle_asciiz)
    - [PRTS 代码](#prts-代码)
    - [AssemblerError](#assemblererror)
    - [symbols.rs](#symbolsrs)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 即将完成 (Home Stretch)

这篇文章将会更长一些。在这篇文章中，我们将完成我们的两遍遍历汇编器，添加一个用于打印的 `PRTS` 操作码，并整理一些其他的松散末端。到最后，我们应该拥有一个具有相当功能的强大解释器和一个简单的汇编语言。当然，我们距离完成这个项目还很远。我想先进行一些不同的教程系列，然后我们将继续进行 `Iridium`。

### 快速更改

在你的 `Cargo.toml` 中添加以下依赖。我们稍后会需要它们：

```toml
log = "0.4"
env_logger = "0.5.13"
byteorder = "1"
```

让我们开始吧……

### 字符串常量

这些是你在代码中定义的字符串。例如，如果你的程序需要频繁打印 "Hello"，你不会想为每次使用都复制一份 "Hello"。相反，我们可以使用与变量非常相似的东西。为了理解这一点，我们需要讨论字节码的一个特殊部分。

### 只读部分

当我们的汇编器写出我们的虚拟机执行的最终字节码块时，它有一个特定的结构，如下：

```assembly
<header>            # <数据头>
<read-only data>    # <只读数据>
<executable data>   # <可执行数据>
```

只读数据是字节码的一个特殊部分，我们在这里存储常量。一个常见的例子是字符串。假设我们像这样声明一个字符串常量：

1. `hello:` 是一个标签
2. `.asciiz` 声明常量的 _类型_
3. `'Hello'` 是常量本身。

当我们的汇编器看到像这样的一行时，它会读取组成 `Hello` 的字节到只读部分的内存中，并添加一个 `0`。由于只读 (`RO`) 部分只是另一个 `Vec<u8>`，它看起来会像这样：

```rust
[72, 101, 108, 108, 111, 0]
```

| | |
|---|---|
| 注意 | 上面的数字是以十进制表示的。在 UTF-8 中，72 == H, 101 == e, 108 == l, 111 == o。|

现在假设我们想要另一个字符串常量，比如 `cat`，它是 `[99, 97, 116]`。如果我们这样写汇编：

```assembly
hello: .asciiz 'Hello'
cat: .asciiz 'cat'
```

然后我们的只读 (`read-only`) 部分会看起来像这样：

```assembly
[72, 101, 108, 108, 111, 0, 99, 97, 116, 0]
```

看到我们是如何将所有常量粘合 (`stick`) 在一起放入一个巨大的向量中的吗？这种方式解答了我们的存储问题，但是当程序执行时，我们该如何检索它们，比如程序遇到了这样的指令：

```assembly
PRTS @hello
```

在我们的汇编器的第一遍遍历期间，它执行了以下操作：

1. 它找到了 `hello:` 并验证了它在符号表中没有相应的条目

2. 它识别到指令是 `.asciiz`，这意味着将下一个操作数视为以 null 结尾的字符串

3. 它解析出 `Hello`，去除了单引号

4. 秘诀在于：它记录下了字符串常量在符号表中的起始偏移量

5. 要检索字符串常量，我们通过查询符号表，并从该点开始读取字节，直到我们遇到一个 `0`

6. `PRTS` 指令知晓需在`只读区域`中进行查找

这正是我所了解的所有以 null 终止的字符串系统的运作机制。另一种处理字符串常量的方法是（令人惊讶的）非 null 终止字符串。您可以同时存储常量的长度，或是其结束位置。这些方法同样可行，但我觉得它们在操作上更为复杂。因此，我们选择了以 null 终止的字符串。=)

对于字符串常量，还有几个其他规则：

1. 用户必须在 `.data` 段中声明它们

2. 所有字符串默认为 UTF-8，并以 `0` 作为终止符

3. 声明字符串常量的格式是：`my_string: .asciiz '<string>'`

4. 在 `.code` 段中，开发者可以使用 `@my_string` 作为操作数

| | |
|---|---|
| 注意 | 您可能会好奇，为何选择了单引号？实不相瞒，尝试让 `nom` 解析单个双引号 `"` 令我颇感头疼。将来，我会研究出如何使其能够兼容单引号 `'` 和双引号 `"` 的解析。|

| | |
|---|---|
| 注意 | 为什么 `.asciiz` 结尾有 `z`，你问？在 MIPS 汇编中，我们无耻地复制的，有一个第二指令，`.ascii`。这声明了一个非空字符终止的字符串。由于 MIPS 汇编是我们的指南，或多或少，我们遵循它的惯例。|

## 更多解析器

前往 `src/assembler/operand_parsers.rs`，让我们为字符串编写一个解析器：

```rust
named!(irstring<CompleteStr, Token>,
    do_parse!(
        tag!("'") >>
        content: take_until!("'") >>
        tag!("'") >>
        (
            Token::IrString{ name: content.to_string() }
        )
    )
);
```

| | |
|---|---|
| 注意 | 你问到它为什么被称为 IrString？为了不与 Rust 关键字冲突。Ir 代表 Iridium，所以 IridiumString。|

我们还需要在 `operand` 解析器中的子解析器列表中添加 `irstring`，如下所示：

```rust
named!(pub operand<CompleteStr, Token>,
    alt!(
        integer_operand |
        label_usage |
        register |
        irstring
    )
);
```

### 以及一个测试

```rust
#[test]
fn test_parse_string_operand() {
    let result = irstring(CompleteStr("'This is a test'"));
    assert_eq!(result.is_ok(), true);
}
```

### 指令

记住，我们使用的是指令，而不是操作码。指令是这种形式：`label: .asciiz 'String content'`。快速查看我们的 `directive_combined` 解析器，揭示我们不允许可选标签：

```rust
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
```

| | |
|---|---|
| 重要 | `directive_combined` 中有一个错误。注意我们有 `tag!(".")` 然后是 `directive_declaration`，它也有一个 `tag!(".")`。净效果是我们正在寻找 `..`。让我们去掉这一个。|

我们可以这样添加一个快速的 `opt!`：

```rust
use assembler::label_parsers::label_declaration;

named!(directive_combined<CompleteStr, AssemblerInstruction>,
    ws!(
        do_parse!(
            l: opt!(label_declaration) >>
            name: directive_declaration >>
            o1: opt!(operand) >>
            o2: opt!(operand) >>
            o3: opt!(operand) >>
            (
                AssemblerInstruction{
                    opcode: None,
                    directive: Some(name),
                    label: l,
                    operand1: o1,
                    operand2: o2,
                    operand3: o3,
                }
            )
        )
    )
);
```

以及它的测试……

```rust
#[test]
fn test_string_directive() {
    let result = directive_combined(CompleteStr("test: .asciiz 'Hello'"));
    assert_eq!(result.is_ok(), true);
    let (_, directive) = result.unwrap();

    // 是的，这就是结果应该是什么
    let correct_instruction =
        AssemblerInstruction {
            opcode: None,
            label: Some(
                Token::LabelDeclaration {
                    name: "test".to_string()
                }),
            directive: Some(
                Token::Directive {
                    name: "asciiz".to_string()
                }),
            operand1: Some(Token::IrString { name: "Hello".to_string() }),
            operand2: None,
            operand3: None };

    assert_eq!(directive, correct_instruction);
}
```

## 终于

好的，我认为我们现在准备好尝试一个包含字符串声明的测试程序了。简单的东西，比如：

```assembly
.data
hello: .asciiz 'Hello everyone!'
.code
hlt
```

等等！我有个好主意！让我们把它作为一个测试放在 `program_parsers` 中！=)

```rust
#[test]
fn test_complete_program() {
    let test_program = CompleteStr(".data\nhello: .asciiz 'Hello everyone!'\n.code\nhlt");
    let result = program(test_program);
    assert_eq!(result.is_ok(), true);
}
```

## 汇编器

解析器正在解析，现在我们需要教汇编器如何处理字符串常量。汇编器将维护一个 `Vec<u8>`。当它发现一个字符串常量时，它会将每个字符转换为字节并追加到只读数据中。当没有更多字符时，它会在末尾添加 `0`（这是 **数值** (`numerical`) 零，**不是** ASCII 或 UTF-8 字符串 "0"！）。它还会记录该常量开始的偏移量并将其放在符号表中。

| | |
|---|---|
| 重要 | 尽管我们正在使用 UTF-8，你应该养成将 UTF 编码视为占用 1 _或更多_ 字节的习惯。毕竟，还有 UTF-16 和 UTF-32。|

## 两遍遍历汇编器

我想如果我直接向你展示完整的两遍遍历汇编器并逐一讲解会更简单。这是声明：

```rust
#[derive(Debug, Default)]
pub struct Assembler {
    /// 跟踪汇编器处于哪个阶段
    phase: AssemblerPhase,
    /// 常量和变量的符号表
    pub symbols: SymbolTable,
    /// 只读数据部分常量放入的地方
    pub ro: Vec<u8>,
    /// 从汇编指令生成的编译字节码
    pub bytecode: Vec<u8>,
    /// 跟踪只读部分当前偏移量
    ro_offset: u32,
    /// 我们在代码中看到的所有部分的列表
    sections: Vec<AssemblerSection>,
    /// 汇编器当前处于的部分
    current_section: Option<AssemblerSection>,
    /// 汇编器当前正在转换为字节码的指令
    current_instruction: u32,
    /// 我们沿途发现的任何错误。最后，我们将它们呈现给用户。
    errors: Vec<AssemblerError>
}
```

我不确定我们是否真的需要保留 `ro_offset` 和 `sections`。

### impl 块

让我们开始逐一实现我们的汇编器函数。我将采用自顶向下的方法，从最高层次的函数开始，然后逐步深入到更小的函数。

### assemble()

这是大多数事物会调用的公共函数，如果他们想把原始字符串变成字节码。`raw` 参数是整个程序的字符串格式。它返回要么是 `Vec<u8>`，要么是 `Vec<AssemblerError>`。

| | |
|---|---|
| 重要 | 当你阅读这段代码时，你可能会想为什么我们没有返回许多值。这是因为当汇编器解析代码时，它正在操纵自己的内部数据结构，比如向 `ro` 添加数据。我之所以这样做，是因为我觉得这似乎更适合 Rust 的所有权系统，而不是试图四处传递引用。|

```rust
pub fn assemble(&mut self, raw: &str) -> Result<Vec<u8>, Vec<AssemblerError>> {
    // 将原始程序通过我们的 `nom` 解析器运行
    match program(CompleteStr(raw)) {
        // 如果没有解析错误，我们现在有 `Vec<AssemblyInstructions>` 要处理。
        // `remainder` _应该_ 是 ""。
        // TODO: 添加对 `remainder` 的检查，确保它是 ""。
        Ok((_remainder, program)) => {
            // 首先获取头部，这样我们就可以稍后将其压入字节码中
            let mut assembled_program = self.write_pie_header();

            // 开始处理 AssembledInstructions。这是我们两遍遍历汇编器的第一遍。
            // 我们向下传递一个只读引用到另一个函数。
            self.process_first_phase(&program);

            // 如果我们在第一遍中累积了任何错误，返回它们，不要尝试进行第二遍
            if !self.errors.is_empty() {
                // TODO: 我们能在这里避免克隆吗？
                return Err(self.errors.clone());
            };

            // 确保我们至少有一个数据段和一个代码段
            if self.sections.len() != 2 {
                // TODO: 详细说明缺少哪一个
                println!("没有找到至少两个部分。");
                self.errors.push(AssemblerError::InsufficientSections);
                // TODO: 我们能在这里避免克隆吗？
                return Err(self.errors.clone());
            }
            // 运行第二遍，将操作码和相关操作数翻译成字节码
            let mut body = self.process_second_phase(&program);

            // 将头部与填充的主体向量合并
            assembled_program.append(&mut body);
            Ok(assembled_program)
        },
        // 如果有解析错误、语法错误等，这个分支将被执行
        Err(e) => {
            println!("解析代码时出错：{:?}", e);
            Err(vec![AssemblerError::ParseError{ error: e.to_string() }])
        }
    }
}
```

### process_first_phase()

这个函数有两个工作：

1. 标签 _声明_（例如，`name:`）
2. 指令（例如，`.data`）

在第一阶段，我们最关心的是找到所有标签声明并将它们放入符号表，确保一切都在一个段中。我对这个函数做了大量注释，所以请仔细阅读。

| | |
|---|---|
| 重要 | 处理错误时，我选择将它们累积到汇编器的一个列表中，并尽可能多地处理。另一种策略是在第一个错误时退出。我选择这样做是因为我认为让用户一次看到所有错误可能更好，而不是进行繁琐的 (tedious) 修复 - 测试 - 修复 - 重复 ( fix-test-fix-repeat) 循环。|

```rust
/// 运行两遍汇编过程的第一阶段。它寻找标签并将它们放入符号表
fn process_first_phase(&mut self, p: &Program) {
    // 即使在第一阶段我们只关心标签和指令，也要遍历每一条指令
    for i in &p.instructions {
        if i.is_label() {
            // TODO: 是否应该将其分解为另一个函数？是否应该放在 process_label_declaration 中？
            if self.current_section.is_some() {
                // 如果我们已经遇到段标题（例如，.code），那么我们可以
                self.process_label_declaration(&i);
            } else {
                // 如果我们还没有遇到段标题，那么我们在段之外遇到了标签，这是不允许的
                self.errors.push(AssemblerError::NoSegmentDeclarationFound{instruction: self.current_instruction});
            }
        }

        if i.is_directive() {
            self.process_directive(i);
        }
        // 这用于跟踪我们在哪个指令上出错了
        // TODO: 我们真的需要跟踪这个吗？
        self.current_instruction += 1;
    }
    // 这个函数完成后，将阶段设置为第二阶段
    self.phase = AssemblerPhase::Second;
}
```

#### process_label_directive()

当汇编器发现一个指令是标签声明时，我们调用这个函数：

```rust
/// 处理一个标签声明，如：
/// hello: .asciiz 'Hello'
fn process_label_declaration(&mut self, i: &AssemblerInstruction) {
    // 检查标签是否为 None 或 String
    let name = match i.get_label_name() {
        Some(name) => { name },
        None => {
            self.errors.push(AssemblerError::StringConstantDeclaredWithoutLabel{instruction: self.current_instruction});
            return;
        }
    };

    // 检查标签是否已经在使用中（在符号表中有条目）
    // TODO: 有更干净的方法来做这个吗？
    if self.symbols.has_symbol(&name) {
        self.errors.push(AssemblerError::SymbolAlreadyDeclared);
        return;
    }

    // 如果我们来到这里，那它就不是我们之前见过的符号，所以把它放在表中
    let symbol = Symbol::new(name, SymbolType::Label);
    self.symbols.add_symbol(symbol);
}
```

### process_second_phase()

这开始了汇编器的第二遍，并做了很多工作：

```rust
/// 运行汇编器的第二遍
fn process_second_phase(&mut self, p: &Program) -> Vec<u8> {
    // 重新启动指令计数
    self.current_instruction = 0;
    // 我们将把要执行的字节码放在一个单独的 Vec 中，这样我们就可以做一些后处理，然后将其与头部和只读部分合并
    // 例子可以是优化，额外检查，等等
    let mut program = vec![];
    // 与第一阶段一样，只是在第二遍我们关心操作码和指令
    for i in &p.instructions {
        if i.is_opcode() {
            // 操作码知道如何正确地将自己转换为 32 位，所以我们可以直接调用 `to_bytes` 并追加到我们的程序中
            let mut bytes = i.to_bytes(&self.symbols);
            program.append(&mut bytes);
        }
        if i.is_directive() {
            // 在这个阶段，我们可以有指令，但我们在第一阶段关心的不同类型的指令。指令本身可以检查汇编器
            // 在哪个阶段，并决定如何处理它
            self.process_directive(i);
        }
        self.current_instruction += 1
    }
    program
}
```

#### process_directive()

```rust
fn process_directive(&mut self, i: &AssemblerInstruction) {
    // 首先让我们确保我们有一个可解析的名称
    let directive_name = match i.get_directive_name() {
        Some(name) => {
            name
        },
        None => {
            println!("指令有一个无效的名称：{:?}", i);
            return;
        }
    };

    // 现在检查是否有任何操作数。
    if i.has_operand
s() {
        // 如果它确实有操作数，我们需要弄清楚它是哪个指令
        match directive_name.as_ref() {
            // 如果这是操作数，我们正在声明一个以空字符终止的字符串
            "asciiz" => {
                self.handle_asciiz(i);
            }
            _ => {
                self.errors.push(AssemblerError::UnknownDirectiveFound{ directive: directive_name.clone() });
                return;
            }
        }
    } else {
        // 如果没有任何操作数（例如，`.code`），那么我们知道它是一个段头
        self.process_section_header(&directive_name);
    }
}
```

#### process_section_header()

这个小函数只是处理任何新的段声明。

```rust
/// 处理一个段头声明，如：
/// .code
fn process_section_header(&mut self, header_name: &str) {
    let new_section: AssemblerSection = header_name.into();
    // 只允许特定的段名
    if new_section == AssemblerSection::Unknown {
        println!("发现一个未知的段头：{:#?}", header_name);
        return;
    }
    // TODO: 检查我们是否真的需要保留所有看到的段的列表
    self.sections.push(new_section.clone());
    self.current_section = Some(new_section);
}
```

#### handle_asciiz()

当调用这个函数来处理字符串常量的声明时。它有大量的注释，所以请仔细阅读。

```rust
/// 处理一个以空字符终止的字符串声明：
/// hello: .asciiz 'Hello!'
fn handle_asciiz(&mut self, i: &AssemblerInstruction) {
    // 作为一个常量声明，这在第一阶段才有意义
    if self.phase != AssemblerPhase::First { return; }

    // 在这种情况下，operand1 将有我们需要读取到 RO 内存的整个字符串
    match i.get_string_constant() {
        Some(s) => {
            match i.get_label_name() {
                Some(name) => { self.symbols.set_symbol_offset(&name, self.ro_offset); }
                None => {
                    // 这将是一个输入：
                    // .asciiz 'Hello'
                    println!("发现一个没有关联标签的字符串常量！");
                    return;
                }
            };
            // 我们将逐字节地将字符串读入只读部分
            for byte in s.as_bytes() {
                self.ro.push(*byte);
                self.ro_offset += 1;
            }
            // 这是我们用来表示字符串结束的空终止位
            self.ro.push(0);
            self.ro_offset += 1;
        }
        None => {
            // 这只是意味着有人出于某种原因键入了 `.asciiz`
            println!("跟随 .asciiz 的字符串常量为空");
        }
    }
}
```

## PRTS 代码

现在我们已经在只读部分存储了东西，能够读取它将会很方便。所以让我们添加一个新的操作码并这样实现：

```rust
Opcode::PRTS => {
    // PRTS 需要一个操作数，要么是字节码的只读部分中的起始索引
    // 或者是一个符号（以 @symbol_name 的形式），它将在符号表中查找偏移量。
    // 这条指令然后读取每个字节并打印它，直到它遇到一个 0x00 字节，这表示字符串的终止
    let starting_offset = self.next_16_bits() as usize;
    let mut ending_offset = starting_offset;
    let slice = self.ro_data.as_slice();
    // TODO: 找到一个更好的方法来做这个。也许我们可以存储字节长度而不是空终止？或者某种形式的缓存，我们在 VM 启动时就通过整个 ro_data 并找到每个字符串及其结束字节位置？
    while slice[ending_offset] != 0 {
        ending_offset += 1;
    }
    let result = std::str::from_utf8(&slice[starting_offset..ending_offset]);
    match result {
        Ok(s) => { print!("{}", s); }
        Err(e) => { println!("为 prts 指令解码字符串时出错：{:#?}", e) }
    };
}
```

## AssemblerError

你可能注意到我们使用了自定义错误类型。我在本教程的这部分开始使用它们，因为项目已经变得足够大和复杂，我认为我们需要它。与其粘贴整个文件内容，不如指向仓库中的文件，因为文章已经快到 3000 字了。=)

你可以在这里找到 `AssemblerError` 模块：<https://gitlab.com/subnetzero/iridium/blob/master/src/assembler/assembler_errors.rs>

| | |
|---|---|
| 注意 | 使用 AssemblerError 将需要将一些 match 分支从 `Some(whatever)` 更改为 `Ok(whatever)`。|

## symbols.rs

您可能会发现，我已经将符号表的相关代码独立出来，放置在单独的文件 src/assembly/symbols.rs 中。您可以通过以下链接一睹其貌：<https://gitlab.com/subnetzero/iridium/blob/master/src/assembler/symbols.rs。>

## 结束

在这篇教程中，我并未涵盖我所做的一些其他微调和更改。如果您的版本由于某些原因无法正常工作，我建议您从仓库中拉取一份全新的副本。按照本教程的内容，正确的代码已在 GitLab 上标记为 0.0.16。

这个系列的文章在这里暂停几周是个不错的节点。我想开始另一个教程系列。我在 Iridium 代码库中到处留下了 `// TODO: <stuff>` 的注释，并且我会尝试在 GitLab 问题跟踪器中为每一项开设工单。如果您愿意贡献 **任何内容**（代码、拼写/语法更正、更符合 Rust 风格的做法，或者任何其他事项），请随时发起合并请求，并在需要帮助或有问题时@我。

我之所以要暂时离开这个项目，另一个原因是 我需要思考接下来要做的事情。我们可以继续语言方面的工作，在现有基础上构建一个更高级的、类似 Python 的语言。或者我们可以专注于虚拟机本身，开始添加并行性和集群等特性。如果您对此有任何想法或建议，欢迎您发表意见或给我发送电子邮件。

希望您喜欢这个系列，并期待下一个系列！

### 原文链接及作者信息

原文链接：[String Constants and More - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-16/)

作者名称：Fletcher

### 成语及典故

成语 & 典故：`A Faustian Bargain`：`浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 只读区域 (Read-Only Region)：指程序运行时，程序不能修改的内存区域。
- 堆栈 (Stack)：指程序运行时，程序临时存储数据的内存区域。
- 堆 (Heap)：指程序运行时，程序动态分配内存的区域。

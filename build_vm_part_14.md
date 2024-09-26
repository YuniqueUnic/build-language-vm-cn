# 14-符号表 - 你想构建一个语言虚拟机吗？

- [14-符号表 - 你想构建一个语言虚拟机吗？](#14-符号表---你想构建一个语言虚拟机吗)
    - [汇编器结构](#汇编器结构)
        - [步骤 1：解析](#步骤-1解析)
        - [步骤 2：符号和表格](#步骤-2符号和表格)
    - [以及更多的测试](#以及更多的测试)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 汇编器结构

欢迎回来！当我们上次离开我们勇敢的读者时，我们正要编写一个汇编器结构。

但是 **为什么** 呢？它做了什么？

这一切究竟何意？！为何感叹问号（`interrobangs`）未能博得更多人的喜爱呢？！嗯哼~

迄今为止，我们一直是将字符串直接交由程序解析器处理。但现在我们谈及的是那些需要保持状态的操作，例如多次遍历。这将作为我们在这个复杂操作之上构建的一个简单抽象。在 `src/assembler/mod.rs` 文件中，请添加以下内容：

```rust
// #[derive(Debug)]
// pub struct Assembler {
//     phase: AssemblerPhase,
// }

#[derive(Debug)]
pub struct Assembler {
    pub phase: AssemblerPhase,
    pub symbols: SymbolTable
}

impl Assembler {
    pub fn new() -> Assembler {
        Assembler {
            phase: AssemblerPhase::First,
            symbols: SymbolTable::new()
        }
    }
}
```

| | |
|---|---|
| 重要 | 不必过于忧虑 `symbols` 字段。我们很快就会着手处理。|

简易汇编器已然成型！请注意我们是如何通过枚举来追踪其阶段的。我们本可以使用 `u8` 或其他类型，但采用这种方式更能发挥 Rust 类型系统的优势，使得代码更加清晰。我们的汇编器将承担以下职责：

1. 将原始字符串交由解析器处理
2. 构建符号表
3. 输出一个 `Vec<u8>`，这是虚拟机能够读取的最终字节码。

### 步骤 1：解析

为此，我们将向汇编器添加几个函数：

```rust
pub fn assemble(&mut self, raw: &str) -> Option<Vec<u8>> {
    match program(CompleteStr(raw)) {
        Ok((_remainder, program)) => {
            self.process_first_phase(&program);
            Some(self.process_second_phase(&program))
        },
        Err(e) => {
            println!("在汇编代码时出错：{:?}", e);
            None
        }
    }
}

fn process_first_phase(&mut self, p: &Program) {
    self.extract_labels(p);
    self.phase = AssemblerPhase::Second;
}

fn process_second_phase(&mut self, p: &Program) -> Vec<u8> {
    let mut program = vec![];
    for i in &p.instructions {
        let mut bytes = i.to_bytes(&self.symbols);
        program.append(&mut bytes);
    }
    program
}
```

新增了三个函数！多么丰富的成果！

以下是发生的过程：

1. `assemble` 函数接受一个原始字符串引用 (`raw string reference`)

2. 汇编器将原始文本 (`raw text`) 传递给 `program` 解析器

3. 它使用 `match` 语句检查程序是否已正确解析

4. 假设它确实正确解析了，我们将程序通过每个汇编器阶段进行处理

5. 汇编器阶段被分解成其他函数，以保持代码的整洁

6. 第一阶段提取所有标签并构建符号表

7. 然后切换到第二阶段

8. 接下来调用第二阶段，它只是在每个 `AssemblerInstruction` 上调用 `to_bytes` 方法

9. 所有字节都被添加到 `Vec<u8>` 中，包含完全汇编的字节码

接下来，让我们看看 `extract_labels` 函数：

```rust
fn extract_labels(&mut self, p: &Program) {
    let mut c = 0;
    for i in &p.instructions {
        if i.is_label() {
            match i.label_name() {
                Some(name) => {
                    let symbol = Symbol::new(name, SymbolType::Label, c);
                    self.symbols.add_symbol(symbol);
                },
                None => {}
            };
        }
        c += 4;
    }
}
```

这个函数的作用是遍历每条指令，寻找标签 _声明_ (_`declarations`_)。也就是说，用户输入了 `some_name: <opcode> …​` 的地方。当它找到一个时，就将其添加到我们的符号表中，以及我们找到标签的字节。

### 步骤 2：符号和表格

我们需要制作三个更多的数据结构：`Symbol`、`SymbolType` 和 `SymbolTable`。将这些放在 `src/assembler/mod.rs` 中。`Symbol` 和 `SymbolType` 看起来像这样：

```rust
#[derive(Debug)]
pub struct Symbol {
    name: String,
    offset: u32,
    symbol_type: SymbolType,
}

impl Symbol {
    pub fn new(name: String, symbol_type: SymbolType, offset: u32) -> Symbol {
        Symbol{
            name,
            symbol_type,
            offset
        }
    }
}

#[derive(Debug)]
pub enum SymbolType {
    Label,
}
```

以后，我们会有更多的 SymbolTypes。现在，我们从 Label 开始。`SymbolTable` 看起来像这样：

```rust
#[derive(Debug)]
pub struct SymbolTable {
    symbols: Vec<Symbol>
}

impl SymbolTable {
    pub fn new() -> SymbolTable {
        SymbolTable{
            symbols: vec![]
        }
    }

    pub fn add_symbol(&mut self, s: Symbol) {
        self.symbols.push(s);
    }

    pub fn symbol_value(&self, s: &str) -> Option<u32> {
        for symbol in &self.symbols {
            if symbol.name == s {
                return Some(symbol.offset);
            }
        }
        None
    }
}
```

| | |
|---|---|
| 重要 | 这最好实现为哈希表 (`HashTable`)。我们稍后会更改它。|

现在我们需要基本的函数（添加、获取符号值），但不用担心，它将会增长。=)

## 以及更多的测试

哈，你以为我忘了，是吧？没那么简单！

这将处理与符号相关的测试：

```rust
#[test]
fn test_symbol_table() {
    let mut sym = SymbolTable::new();
    let new_symbol = Symbol::new("test".to_string(), SymbolType::Label, 12);
    sym.add_symbol(new_symbol);
    assert_eq!(sym.symbols.len(), 1);
    let v = sym.symbol_value("test");
    assert_eq!(true, v.is_some());
    let v = v.unwrap();
    assert_eq!(v, 12);
    let v = sym.symbol_value("does_not_exist");
    assert_eq!(v.is_some(), false);
}
```

以及对于汇编器：

```rust
#[test]
fn test_assemble_program() {
    let mut asm = Assembler::new();
    let test_string = "load $0 #100\nload $1 #1\nload $2 #0\ntest: inc $0\nneq $0 $2\njmpe @test\nhlt";
    let program = asm.assemble(test_string).unwrap();
    let mut vm = VM::new();
    assert_eq!(program.len(), 21);
    vm.add_bytes(program);
    assert_eq!(vm.program.len(), 21);
}
```

## 结束

我们将这部分称为好。[维基百科](https://en.wikipedia.org/wiki/Symbol_table)上有一篇关于 `符号表` 的好文章。如果一开始它们令人困惑，不用担心。接下来，我们将使用 `clap` 来为我们的 VM 制作一个更好的 CLI 界面。

### 原文链接及作者信息

- 原文链接：[Symbol Tables - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-14/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 汇编器（Assembler）：一种程序，用于将汇编语言代码转换成机器代码。
- 符号表（Symbol Table）：汇编器在处理代码时维护的数据结构，用于存储有关代码的元数据，如符号的字节偏移量。
- 标签（Label）：在汇编语言中用于标记特定位置的标识符。
- 指令（Directive）：汇编语言中用于控制汇编器行为的特殊指令。
- 操作码（Opcode）：用于表示不同操作的代码。
- 寄存器（Register）：用于存储数据的硬件元素，常用于编程中的变量存储。
- 整数操作数（Integer Operand）：指令中表示整数数据的部分。
- 程序（Program）：一系列指令的集合，用于执行特定的任务或操作。
- 向量（Vector）：一种数据结构，用于存储一系列元素。
- 十六进制（Hexadecimal）：一种基数为 16 的计数系统，使用数字 0-9 和字母 A-F 表示值。

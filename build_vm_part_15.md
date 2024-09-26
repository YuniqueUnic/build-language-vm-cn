# 15-汇编器 CLI 改进 - 你想构建一个语言虚拟机吗？

- [15-汇编器 CLI 改进 - 你想构建一个语言虚拟机吗？](#15-汇编器-cli-改进---你想构建一个语言虚拟机吗)
    - [命令行界面（CLI）改进](#命令行界面cli改进)
        - [添加依赖](#添加依赖)
        - [使用 `clap`](#使用-clap)
        - [cli.yml](#cliyml)
        - [Makefile](#makefile)
        - [EPIE Header](#epie-header)
    - [虚拟机](#虚拟机)
        - [未能通过的测试 (Broken Tests)](#未能通过的测试-broken-tests)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 命令行界面（CLI）改进

必须启动解释器，然后输入 `.load_file` 等操作是相当繁琐的。让我们改进一下，使得虚拟机可以直接尝试执行作为参数传递给它的文件。在 Rust 中有一个非常方便的 crate（库）叫做 `clap`，它可以让我们轻松实现这个功能。

我们想要的行为是：

1. 如果用户只输入 `iridium` 而没有其他内容，他们将直接进入 `REPL` 环境
2. 如果用户输入 `iridium /path/to/valid/*.iasm`，也就是说，他们想要直接执行一个代码文件，那么程序应该执行该文件然后退出

| | |
|---|---|
| 重要 | 如果你想要能够从任何地方运行 iridium 可执行文件，你可能需要将其放在 `/usr/local/bin` 或你 PATH 路径中的其他位置。|

### 添加依赖

在 `Cargo.toml` 文件中添加：

```toml
clap = { version = "2.32", features = ["yaml"] }
```

要了解所有关于 [clap](https://crates.io/crates/clap#quick-example) 的信息，我建议阅读他们的官方网站。它写得非常好，应该能覆盖我们所需要的一切。如果你不想看，知道 `clap` 是一个工具，可以轻松编写像典型命令行界面（CLI）命令一样工作的应用程序：标志、帮助等。

### 使用 `clap`

首先，在 `main.rs` 文件顶部添加以下内容：

```rust
extern crate clap;
use clap::{Arg, App, SubCommand};
```

接下来，我们将 `REPL` 启动代码提取出来。在 `main.rs` 中创建这个函数：

```rust
/// 启动一个 `REPL`，直到用户终止它
fn start_repl() {
    let mut repl = repl::REPL::new();
    repl.run();
}
```

我们需要一个从文件读取数据的函数。将其放在 `main.rs` 中：

```rust
/// 尝试读取文件并返回内容。如果由于任何原因无法读取文件，则退出。
fn read_file(tmp: &str) -> String {
    let filename = Path::new(tmp);
    match File::open(Path::new(&filename)) {
        Ok(mut fh) => {
            let mut contents = String::new();
            match fh.read_to_string(&mut contents) {
                Ok(_) => {
                    return contents;
                },
                Err(e) => {
                    println!("读取文件时出错：{:?}", e);
                    std::process::exit(1);
                }
            }
        },
        Err(e) => {
            println!("文件未找到：{:?}", e);
            std::process::exit(1)
        }
    }
}
```

这些都是相当标准的模板 (`boilerplate`) 代码。下面是新的 `main()` 函数和导入：

```rust
use std::path::Path;
use std::fs::File;
use std::io::prelude::*;

#[macro_use]
extern crate nom;

#[macro_use]
extern crate clap;

use clap::App;

pub mod assembler;
pub mod instruction;
pub mod repl;
pub mod vm;

fn main() {
    let yaml = load_yaml!("cli.yml");
    let matches = App::from_yaml(yaml).get_matches();
    let target_file = matches.value_of("INPUT_FILE");
    match target_file {
        Some(filename) => {
            let program = read_file(filename);
            let mut asm = assembler::Assembler::new();
            let mut vm = vm::VM::new();
            let program = asm.assemble(&program);
            match program {
                Some(p) => {
                    vm.add_bytes(p);
                    vm.run();
                    std::process::exit(0);
                },
                None => {}
            }
        },
        None => {
            start_repl();
        }
    }
}
```

如果用户键入 `iridium /something.iasm`，那么将尝试：

1. 从该文件读取数据
2. 将其通过我们的 `nom` 解析器
3. 如果成功，将其添加到虚拟机，运行它，然后退出并返回 0

如果他们键入 `iridium`，则启动 `REPL`。

### cli.yml

我选择将配置放在 `src/cli.yml` 中，但你可以将其放在其他地方，或者使用不同的配置选项。我的示例如下：

```yaml
name: iridium
version: "0.0.1"
author: Fletcher Haynes <fletcher@subnetzero.io>
about: Iridium 语言解释器
args:
    - INPUT_FILE:
        help: 要运行的 .iasm 或 .ir 文件的路径
        required: false
        index: 1
```

但 `clap` 允许使用另外两种方法：代码和宏。如果你愿意，可以自定义。

### Makefile

我在仓库中留下了我的 Makefile，所以你可以使用它来帮助将二进制文件安装到 `/usr/local/bin/` 中，如果你愿意的话。

### EPIE Header

由于这部分还剩一些文字，让我们教我们的汇编器如何写出 PIE 头。在 `src/assembler/mod.rs` 的顶部添加一些必要的常量：

```rust
const PIE_HEADER_PREFIX: [u8; 4] = [45, 50, 49, 45];
const PIE_HEADER_LENGTH: usize = 64;
```

在 `Assembler` 的 `impl` 中添加这个函数：

```rust
fn write_pie_header(&self) -> Vec<u8> {
    let mut header = vec![];
    for byte in PIE_HEADER_PREFIX.into_iter() {
        header.push(byte.clone());
    }
    while header.len() <= PIE_HEADER_LENGTH {
        header.push(0 as u8);
    }
    header
}
```

这将输出我们的头文件，目前是 4 个字节和 60 个 0。重要的是要填充头文件，以便我们以后可以使用这些字节。`Assembler` `impl` 中我们需要调整的最后一个函数是 `assemble`，以便它实际使用头文件生成器：

```rust
// 别忘了在文件顶部也添加这个
use assembler::PIE_HEADER_PREFIX;

pub fn assemble(&mut self, raw: &str) -> Option<Vec<u8>> {
    match program(CompleteStr(raw)) {
        Ok((_remainder, program)) => {
            // 首先获取头文件，以便我们可以将其压缩到字节码中
            let mut assembled_program = self.write_pie_header();
            self.process_first_phase(&program);
            let mut body = self.process_second_phase(&program);

            // 将头文件与填充的主体向量合并
            assembled_program.append(&mut body);
            Some(assembled_program)
        },
        Err(e) => {
            println!("汇编代码时出错：{:?}", e);
            None
        }
    }
}
```

现在如果你运行 `cargo test`，你会注意到 `test_bytes_assemble` 失败了。这是因为我们增加了 64 个字节。暂时将该测试中的 28 改为 92。

## 虚拟机

当然，我们的虚拟机还不知道要查找头文件。我们可以在其中添加一个 `verify_header` 函数。在 `src/vm.rs` 中，将其添加到 `VM` 的 `impl` 中：

```rust
/// 处理虚拟机想要执行的字节码的头文件
fn verify_header(&self) -> bool {
    if self.program[0..4] != PIE_HEADER_PREFIX {
        return false;
    }
    true
}
```

最后，我们将虚拟机的 `Program Counter` 设置为 65，因为我们目前还不需要处理头文件。

### 未能通过的测试 (Broken Tests)

这些更改将破坏 `vm.rs` 中使用 `vm.run()` 而不是 `vm.run_once()` 的任何测试。为什么？因为 `vm.run()` 现在包含头文件验证 (`header validation`)，但 `run_once()` 不检查头文件。

我在测试模块中写了一个简单的函数来预处理头文件：

```rust
fn prepend_header(mut b: Vec<u8>) -> Vec<u8> {
    let mut prepension = vec![];
    for byte in PIE_HEADER_PREFIX.into_iter() {
        prepension.push(byte.clone());
    }
    while prepension.len() <= PIE_HEADER_LENGTH {
        prepension.push(0);
    }
    prepension.append(&mut b);
    prepension
}
```

在测试中这样使用它：

```rust
#[test]
fn test_mul_opcode() {
    let mut test_vm = get_test_vm();
    test_vm.program = vec![3, 0, 1, 2];
    test_vm.program = prepend_header(test_vm.program);
    test_vm.run();
    assert_eq!(test_vm.registers[2], 50);
}
```

## 结束

我认为我们已经准备好进行两遍遍历的工作流程，所以我们将在下一节开始！

### 原文链接及作者信息

原文链接：[Assembler CLI Improvements - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-15/)

作者名称：Fletcher

### 成语及典故

成语 & 典故：`A Faustian Bargain`：`浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- Makefile: 是一个特殊类型的文件，它用于自动化构建过程，特别是在编译软件时。在 Unix 和类 Unix 系统中，make 工具会使用 Makefile 来执行一系列的任务，比如编译源代码、运行测试、创建文档、安装软件等。
- Broken Tests: 指的是在软件开发过程中，那些未能通过的单元测试或集成测试。这些测试原本是为了验证代码的正确性和稳定性，但当它们失败时，通常意味着代码中存在以下问题：
- 符号表（Symbol Table）：汇编器在处理代码时维护的数据结构，用于存储有关代码的元数据，如符号的字节偏移量。

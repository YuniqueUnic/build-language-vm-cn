# 21-头部偏移 - 你想构建一个语言虚拟机吗？

- [21-头部偏移 - 你想构建一个语言虚拟机吗？](#21-头部偏移---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [问题](#问题)
    - [修复方法](#修复方法)
        - [计算长度](#计算长度)
            - [另外一件事](#另外一件事)
        - [读取偏移量](#读取偏移量)
        - [测试](#测试)
    - [结束](#结束)

## 引言

你好！在本文中，我们将修复一个小 (`teensy`) 错误。=)

## 问题

Iridium 程序（字节码）始终以 64 字节的头部开始。前 4 个字节是一个常量魔数，让 Iridium 虚拟机知道这是 Iridium 字节码。其余的都是零。一个例子是：

然后是程序的只读部分，包含像常量这样的东西。我们在组装程序之前不知道这一部分的长度。如果有人声明了字符串 "Hello" 的常量，只读部分将如下所示：

```assembly
[72, 101, 108, 108, 111, 0]
```

在我们的程序中，我们现在有 64（头部长度）+ 6（常量 Hello，记住字符串以空字符终止）字节，总共 70 字节。虚拟机需要从程序的第 71 个字节开始执行。

## 修复方法

不用担心，修复这个问题非常简单！我们需要：

> 1. Calculate the final length of the read-only section during assembly
> 2. Write that to the header using the next 4 bytes
> 3. Initialize our VM with that PC

1. 在组装过程中计算只读部分的最终长度。

2. 使用接下来的 4 个字节将该长度写入头部。

3. 使用该 PC 初始化我们的虚拟机。

### 计算长度

前往 `src/assembler/mod.rs` 并让我们修改一个函数。我们将使用 byteorder crate 的便捷 (`nifty`) 特性来完成此操作。

我们要修改的函数是 `write_pie_header`。将其更改为：

```rust
fn write_pie_header(&self) -> Vec<u8> {
    let mut header = vec![];
    for byte in &PIE_HEADER_PREFIX {
        header.push(byte.clone());
    }

    // 现在我们需要计算起始偏移量，以便虚拟机知道只读部分在哪里结束

    // 首先我们声明一个空向量，让 byteorder 写入
    let mut wtr: Vec<u8> = vec![];

    // 将只读部分的长度写入向量，并将其转换为 u32
    // 这很重要，因为 byteorder crate 将按需用零填充
    wtr.write_u32::<LittleEndian>(self.ro.len() as u32).unwrap();

    // 将这 4 个字节直接追加到头部的前四个字节后面
    header.append(&mut wtr);

    // 现在填充字节码头部的其余部分
    while header.len() < PIE_HEADER_LENGTH {
        header.push(0 as u8);
    }

    header
}
```

中间的三行新代码是关键；我为每一行都添加了注释，解释了它的功能。

不要忘记添加：

```rust
use byteorder::{LittleEndian, WriteBytesExt};
```

到你的 `src/assembler/mod.rs` 文件中。

#### 另外一件事

我们需要确保我们在设置所有只读数据后才写入头部。在 `src/assembler/mod.rs` 中的 `assemble` 函数中，将对 `write_pie_header` 的调用移动到主体生成后，如下所示：

```rust
let mut body = self.process_second_phase(&program);

// Get the header so we can smush it into the bytecode letter
// 获取头部，以便我们可以将其压缩进字节码
let mut assembled_program = self.write_pie_header();

// Merge the header with the populated body vector
// 将头部与填充的主体向量合并
assembled_program.append(&mut body);
```

### 读取偏移量

现在我们需要教我们的虚拟机如何读取偏移量。在 `src/vm.rs` 中，添加以下函数：

```rust
fn get_starting_offset(&self) -> usize {
    // 我们只想读取紧接魔数 (the magic number) 之后的 4 个字节的切片
    let mut rdr = Cursor::new(&self.program[4..8]);
    // 将其读取为 u32，转换为 usize（因为虚拟机的 PC 属性是 usize），然后返回
    rdr.read_u32::<LittleEndian>().unwrap() as usize
}
```

然后在虚拟机的 `run` 函数中，将：

```rust
self.pc = 64;
```

替换为：

```rust
self.pc = 64 + self.get_starting_offset();
```

### 测试

现在让我们编写一个测试来确保它有效！在 `src/assembler/mod.rs` 中，添加此测试：

```rust
#[test]
/// 简单测试只读部分的数据
fn test_code_start_offset_written() {
    let mut asm = Assembler::new();
    let test_string = ".data\ntest1: .asciiz 'Hello'\n.code\nload $0 #100\nload $1 #1\nload $2 #0\ntest: inc $0\nneq $0 $2\njmpe @test\nhlt";
    let program = asm.assemble(test_string);
    assert_eq!(program.is_ok(), true);
    assert_eq!(program[4], 6);
}
```

使用那个测试字符串，我们应该有一个看起来像这样的头部：

```assembly
[45, 50, 49, 45, 6, 0, 0, 0, ... ]
```

如果我们运行我们的测试，我们应该会看到：

```sh
$ cargo test test_code_start_offset_written -- --nocapture
    Finished dev [unoptimized + debuginfo] target(s) in 0.11s
     Running target/debug/deps/iridium-981657ef3cdcfc6e

running 1 test
test assembler::tests::test_code_start_offset_written ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 45 filtered out

     Running target/debug/deps/iridium-87ed8e3d062c1031

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests iridium

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

耶，它有效！

## 结束

这并不难，对吧？你可以在这里查看代码：<https://gitlab.com/subnetzero/iridium/tags/0.0.21>。

下次见！"

原文出处：[Header Offset - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-21/)
作者名：Fletcher

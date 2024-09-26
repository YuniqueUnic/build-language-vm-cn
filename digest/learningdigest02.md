# 02-RUST Virtual Machine Learning

## 今日学习总结

> 当前北京时间：2024 年 09 月 12 日 15:51:24

### 进度

- [x] build_vm_part_10.md
- [x] build_vm_part_11.md
- [x] build_vm_part_12.md
- [x] build_vm_part_13.md
- [x] build_vm_part_14.md
- [x] build_vm_part_15.md

### 笔记

- label 跳转的原理：两遍遍历器实现跳转
因为存在俩个阶段，所以需要两遍遍历器实现跳转。那么也就是需要一个这样的数据结构 (Assembler)：

```rust
// 枚举：第一阶段/第二阶段
#[derive(Debug, PartialEq, Clone)]
pub enum AssemblerPhase { 
    First,
    Second,
}


#[derive(Debug)]
pub struct Assembler {
    pub phase: AssemblerPhase, // 当前阶段：第一阶段/第二阶段
    pub symbols: SymbolTable   // 标签表：存储所有的 label: 标签和行号
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

1. **第一阶段：解析代码**
2. 将一串代码输入之后，尝试对代码进行解析，解析出所有的 label: 标签，并记录下来，同时记录对应后续调用 @label 标签对应的行号。
    1. 因为需要存储所有的 label: 标签，所以需要一个哈希表来存储，同时需要一个数组来存储行号。
    2. 继而就可以构建一个 SymbolTable，用来存储所有的 label: 标签和行号。

```rust
#[derive(Debug, PartialEq, Clone)]
pub struct SymbolTable {
    // 存储所有的 Symbols (Labels)
    pub symbols: Vec<Symbol>,
}
```

    3. 有了 SymbolTable , 就需要考虑一下 Symbol 的数据结构了，要有 name 吧，要有 line 行号吧，额外还要有个 label 的类型，以便于后续拓展。

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum SymbolType {
    Label,
}

#[derive(Debug, PartialEq, Clone)]
pub struct Symbol {
    name: String,
    symbol_type: SymbolType,
    offset: Option<u32>,
}
```

    4. 名字有了，类型也有了，为什么要用 offset 来代替行号 line 呢？因为在第二遍的时候，要跳转，只用偏移量就可以按照需要跳转的位置 + 偏移量的计算方式就可以跳转了。
    5. 如此这般，便在第一次遍历的时候，构建好了 SymbolTable，那么在第二次遍历的时候，就可以根据 SymbolTable 来进行跳转了。
2. **第二阶段：构建虚拟机代码** 
    1. 它只是在每个 AssemblerInstruction 上调用 `to_bytes` 方法
    2. 所有字节都被添加到 `Vec<u8>` 中，包含完全汇编的字节码
.

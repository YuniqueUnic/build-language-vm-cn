# 00-RUST Virtual Machine Learning

## 今日学习总结

> 当前北京时间：2024 年 09 月 11 日 01:12:33

### 进度

- [x] build_vm_part_00.md
- [x] build_vm_part_01.md
- [x] build_vm_part_02.md
- [x] build_vm_part_03.md
- [x] build_vm_part_04.md
- [x] build_vm_part_05.md

### 笔记

**操作码表**

```rust
#[derive(Debug, PartialEq)]
pub enum Opcode {
    LOAD,    // 0
    ADD,     // 1
    SUB,     // 2
    MUL,     // 3
    DIV,     // 4
    HLT,     // 5
    JMP,     // 6
    JMPF,    // 7
    JMPB,    // 8
    EQ,      // 9
    NEQ,     // 10
    GTE,     // 11
    LTE,     // 12
    LT,      // 13
    GT,      // 14
    JMPE,    // 15
    NOP,     // 16
    ALOC,    // 17
    INC,     // 18
    DEC,     // 19
    DJMPE,   // 20
    IGL,     // _
    PRTS,    // 21
    LOADF64, // 22
    ADDF64,  // 23
    SUBF64,  // 24
    MULF64,  // 25
    DIVF64,  // 26
    EQF64,   // 27
    NEQF64,  // 28
    GTF64,   // 29
    GTEF64,  // 30
    LTF64,   // 31
    LTEF64,  // 32
    SHL,     // 33
    SHR,     // 34
    AND,     // 35
    OR,      // 36
    XOR,     // 37
    NOT,     // 38
    LUI,     // 39
    CLOOP,   // 40
    LOOP,    // 41
    LOADM,   // 42
    SETM,    // 43
    PUSH,    // 44
    POP,     // 45
    CALL,    // 46
    RET,     // 47
}

impl From<u8> for Opcode {
    fn from(value: u8) -> Self {
        match value {
            0 => Opcode::LOAD,
            1 => Opcode::ADD,
            2 => Opcode::SUB,
            3 => Opcode::MUL,
            4 => Opcode::DIV,
            5 => Opcode::HLT,
            6 => Opcode::JMP,
            7 => Opcode::JMPF,
            8 => Opcode::JMPB,
            9 => Opcode::EQ,
            10 => Opcode::NEQ,
            11 => Opcode::GTE,
            12 => Opcode::LTE,
            13 => Opcode::LT,
            14 => Opcode::GT,
            15 => Opcode::JMPE,
            16 => Opcode::NOP,
            17 => Opcode::ALOC,
            18 => Opcode::INC,
            19 => Opcode::DEC,
            20 => Opcode::DJMPE,
            21 => Opcode::PRTS,
            22 => Opcode::LOADF64,
            23 => Opcode::ADDF64,
            24 => Opcode::SUBF64,
            25 => Opcode::MULF64,
            26 => Opcode::DIVF64,
            27 => Opcode::EQF64,
            28 => Opcode::NEQF64,
            29 => Opcode::GTF64,
            30 => Opcode::GTEF64,
            31 => Opcode::LTF64,
            32 => Opcode::LTEF64,
            33 => Opcode::SHL,
            34 => Opcode::SHR,
            35 => Opcode::AND,
            36 => Opcode::OR,
            37 => Opcode::XOR,
            38 => Opcode::NOT,
            39 => Opcode::LUI,
            40 => Opcode::CLOOP,
            41 => Opcode::LOOP,
            42 => Opcode::LOADM,
            43 => Opcode::SETM,
            44 => Opcode::PUSH,
            45 => Opcode::POP,
            46 => Opcode::CALL,
            47 => Opcode::RET,
            _ => Opcode::IGL,
        }
    }
}
```

```rust
pub struct VM {
    /// 模拟硬件寄存器的数组
    registers: [i32; 32],
    /// 跟踪正在执行的字节的程序计数器
    pc: usize,
    /// 正在运行的程序的字节码
    program: Vec<u8>,
    /// 包含模除操作的余数
    remainder: usize,
    /// 包含最后一次比较操作的结果
    equal_flag: bool,
}
```

- registers: `[i32; 32]` 这个东西 (`寄存器`) 就是拿来存放实际的值的一个数组，32 个 32 位的整数。
  - 如果需要取用一个暂存的值，那么需要用索引寄存器的方式来取得。
  - 例如：`LOAD $1 #10` 中 `$1` 的 `1` 就是索引， `#10` 的 `10`就是值。即 `registers[1] = 10`
  - 再比如：`ADD $1 $2 $3` `$1` `$2` `$3` 就是索引， `ADD` 就是操作符， `$1` `$2` `$3` 就是操作数。即 `registers[3] = registers[1] + registers[2]`
  - 又比如：`JMP $1` `$1` 就是索引， `JMP` 就是操作符， `$1` 就是操作数。即 `pc = registers[1]`
- pc: `usize` 这个是程序计数器，用来记录当前执行的字节码在 program 数组中的索引。
- program: `Vec<u8>` 这个是程序，用来存储字节码。
  - 例如：`[0, 1, 1, 244, 1, 2, 0, 3]` 在查询了操作码表之后就可以表示为：`LOAD #1 calc(1u16 << 8 +244); ADD $2 $0 #3;`
  - 翻译为代码：`registers[1] = ((1u8 as u16) << 8) + 244 = 500; registers[3] = registers[2] + registers[0];`
- reminder: `Vec<u8>` 这个用来存储部分算术操作码的余数，
  - 例如：`DIV $1 $2 $3` 即为 `registers[3] = registers[1] / registers[2]; VM.remainder = registers[1] % registers[2];`。
- equal_flag: `bool` 用来表示两个数是否相等，
  - 例如：`EQ $1 $2` 即为 `VM.equal_flag = registers[1] == registers[2];` 但是本设计中为了和 MIPS 的 `EQ $1 $2 $3` 指令保持一致，本设计中的 `EQ` 指令的实现为 `VM.equal_flag = registers[1] == registers[2]; VM.next_8_bits();` 使用了 `next_8_bits()` 函数来跳过一个字节。

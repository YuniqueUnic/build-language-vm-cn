# 11-内存 - 你想构建一个语言虚拟机吗？

- [11-内存 - 你想构建一个语言虚拟机吗？](#11-内存---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [计算机内存简介](#计算机内存简介)
        - [操作系统](#操作系统)
        - [堆 (`Heap`)](#堆-heap)
        - [内存泄漏和垃圾回收](#内存泄漏和垃圾回收)
            - [引用计数](#引用计数)
            - [标记和清扫](#标记和清扫)
    - [Iridium 中的内存分配](#iridium-中的内存分配)
        - [汇编中的内存](#汇编中的内存)
            - [表示内存](#表示内存)
    - [为我们的虚拟机添加堆](#为我们的虚拟机添加堆)
    - [分配操作码](#分配操作码)
        - [测试，测试……](#测试测试)
    - [结束](#结束)
        - [原文链接及作者信息](#原文链接及作者信息)
        - [成语及典故](#成语及典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

目前，我们只有一个存储数据的地方：寄存器 (`registers`)。这些非常好，但我们只有 32 个，而且它们只存储 i32 类型的数据。如果我们想要存储字符串呢？或者如果用户想要创建并存储一个大型数据结构呢？我们需要的不仅仅是寄存器。

### 计算机内存简介

很久以前，在人们穿着降落伞裤 (`parachute pants`) 和带有垫肩的连衣裙 (`dresses with shoulderpads`) 的时代，计算机内存是一系列插槽 (`slots`)，任何代码都可以写入任何插槽。例如，你可以通过直接写入控制屏幕上显示内容的内存插槽来进行绘图。但随着时间的推移，计算机变得更加复杂，拥有更多的内存，并开始同时运行多个程序。由于某个不相关的程序向内存插槽写入了错误的值而导致你的程序崩溃，这并不有趣。

如今，从操作系统和 CPU 的角度来看，内存要复杂得多。但对于程序员来说，它看起来仍然是一样的：他们的程序可以独占内存。

### 操作系统

当用户启动应用程序并开始执行时，操作系统在幕后执行各种奇技淫巧 (`trickery`)，以便运行的应用程序可以表现得好像以下两个陈述是真的：

1. 应用程序独占 (`sole access`) 内存访问权。
2. 应用程序拥有无限 (`unlimited`) 的内存。

该程序可以使用系统调用 `malloc`、`free` 等来获取更多内存并归还它。实际上，操作系统协调 (`coordinates`) 所有内存分配，以防止应用程序相互干扰。通过 `malloc` 可用的内存称为……

### 堆 (`Heap`)

堆用于存储在编译时无法预知空间需求的内容。如果我们声明一个大小为 100 的 `f64` 数组，那么编译器确切知道应用程序需要多少内存，并在编译时进行请求。但如果我们想要一个可以无限增长的数组呢？那么我们需要使用堆来每次需要存储更多 `f64` 时请求内存。

想象一个拥有 4GB RAM 的系统。当在其上运行的所有应用程序请求超过 4GB 时会发生什么？这通常是系统开始进行称为 _交换_ (_`swap`_) 的可怕活动的时候。这意味着它开始使用本地磁盘作为额外的 RAM。

| | |
|---|---|
| 注 | 在服务器上禁用交换变得越来越普遍。如果系统不能交换并且内存不足，操作系统通常会选择一个进程并终止它。|

### 内存泄漏和垃圾回收

- 内存溢出 (`memory overflow`) 是指应用程序请求的内存超过系统可用内存。
- 内存泄漏 (`memory leaks`) 是指应用程序在运行时没有释放内存。
- 垃圾回收 (`garbage collection`) 是一种自动处理内存分配和释放的技术。

如果你从未使用过需要显式内存管理的语言（C、C++ 是最常见的），这对你来说可能看起来很奇怪。我们甚至在 Rust 中不必使用 `malloc` 或 `free`。让我们来看看维基百科上的这个 C 代码片段：

```c
#include <stdlib.h>

void function_which_allocates(void) {
    /* 分配一个 45 个浮点数的数组 */
    float *a = malloc(sizeof(float) * 45);

    /* 使用 'a' 的额外代码 */

    /* 返回到 main，忘记了释放我们 malloc'd 的内存 */
}

int main(void) {
    function_which_allocates();

    /* 指针 'a' 不再存在，因此无法释放，
     但内存仍然被分配。发生了泄漏。 */
}
```

看到它如何 `malloc` 但从未 `free` 内存了吗？从操作系统的角度来看，那部分内存仍然属于 C 应用程序；但应用程序一旦 `a` 超出作用域就无法访问它。

大多数高级语言通过 _垃圾回收_ 来处理这个问题。当你在 Python、Ruby、Java、Go 或任何其他具有 GC 的语言中运行程序时，它会跟踪创建的所有数据结构，并在检测到不再使用时回收内存。这使人类免于进行内存记账（我们在这方面真的 **非常** 差），但权衡是它有计算 (`computational`) 开销，也许更关键的是，它会导致应用程序出现非确定性 (`non-deterministic`) 持续时间的暂停。

常见的垃圾收集器有几种类型：引用计数 (`reference counting`)、标记 - 清扫 (`mark and sweep`) 、分代 (`generational`) 和追踪 (`tracing`) 是最常见的一些。现在，我将介绍引用计数和标记 - 清扫。

#### 引用计数

在这种垃圾收集模型中，语言会跟踪有多少东西引用了某个内容。当那个计数达到 0 时，它就知道被引用的内容不再使用，可以被销毁。

看看这段 Python 代码：

```python
def leak():
  class A:
    def __init__(self):
      self.b = None

  class B:
    def __init__(self):
      self.a = None

  a = A()
  b = B()
  a.b = b
  b.a = a

leak()
```

我们在这里创建了一个 _循环依赖_ (_`cyclic dependency`_)：`a` 引用 `b`，`b` 引用 `a`。因为我们没有从函数 `leak` 返回 `a` 或 `b`，一旦那个函数返回，我们就无法访问这些类实例。但这里是重要的部分：**它们相互保持存活**。它们的引用计数永远不会变为零。如果我们只依赖引用计数，这将是一个内存泄漏。

优点是，引用计数很快，因为它只需要增加或减少一个数字。

| | |
|---|---|
| 注 | Rust 有以 [Rc](https://doc.rust-lang.org/std/rc/index.html) 和 [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) 形式的引用计数。Objective-C 和 Swift 也是使用引用计数的语言的例子。|

#### 标记和清扫

这是最简单的可以处理循环依赖的垃圾收集器类型。想象每个程序都有一个根节点，当应用程序运行并创建新的数据结构时，它们成为一棵树的一部分，根节点在底部。标记和清扫算法可以从根节点开始，遍历 (`traverse`) _所有_ 附加到它的内容，并在每个可以到达的内容上设置一个位。这称为标记。

但是，这并不能帮助我们处理与根节点无关的循环依赖。解决方案是我们的虚拟机保留它创建的所有内容的引用。用户/开发者不知道这一点，也无法访问虚拟机的全局数据结构列表。一旦 GC 标记了它可以达到的所有内容，_虚拟机_ 可以遍历它的全局列表并删除所有未标记的内容。

当然，这些类型的 GC 有一个缺点，那就是暂停。如果你曾使用 Java 工作过，你可能经历过令人恼火的暂停，你的整个程序似乎在一些毫秒内停止了。这通常是由于 GC 正在进行操作。很难避免在执行 GC 扫描时暂停执行的需求，但你可以优化以减少暂停持续时间，或减少收集的垃圾数量。

| | |
|---|---|
| 注 | Java 和 Go 是使用更高级垃圾收集器的语言的例子。JVM GC 非常高级，可以根据你的应用程序需求以多种模式运行。Go GC 要简单得多，专注于低延迟 (`low latency`)。|

## Iridium 中的内存分配

在我们为 Iridium 编写垃圾收集器之前，我们需要支持内存的分配与释放。目前，我们的虚拟机模拟的是 CPU 的行为；现在，我们还需要它模拟内存的行为。

### 汇编中的内存

大多数汇编语言提供了一个指令，从操作系统请求内存并释放它；通常是 `brk` 或类似的。我们可以做类似的事情。

#### 表示内存

我们如何在虚拟机中表示内存？当然是用 `Vec<u8>`！然后请求更多内存就成为扩展向量的简单问题。

## 为我们的虚拟机添加堆

目前，我们的虚拟机看起来像这样：

```rust
pub struct VM {
    pub registers: [i32; 32],
    pc: usize,
    pub program: Vec<u8>,
    remainder: usize,
    equal_flag: bool,
}
```

要添加一个堆，我们可以这样做：

```rust
pub struct VM {
    pub registers: [i32; 32],
    pc: usize,
    pub program: Vec<u8>,
    heap: Vec<u8>,
    remainder: usize,
    equal_flag: bool,
}
```

然后在我们的 `impl` 块中：

```rust
pub fn new() -> VM {
    VM {
        registers: [0; 32],
        program: vec![],
        heap: vec![],
        pc: 0,
        remainder: 0,
        equal_flag: false,
    }
}
```

我们现在有一个堆。=)

## 分配操作码

在本教程的最后一件事中，让我们添加一个名为 `ALOC` 的操作码，它通过寄存器中给出的参数扩展堆向量的大小。我相信你还记得如何添加新的操作码，所以我这里不再重复。=)

`ALOC` 指令的处理代码是：

```rust
Opcode::ALOC => {
    let register = self.next_8_bits() as usize;
    let bytes = self.registers[register];
    let new_end = self.heap.len() as i32 + bytes;
    self.heap.resize(new_end as usize, 0);
}
```

### 测试，测试……

```rust
#[test]
fn test_aloc_opcode() {
    let mut test_vm = get_test_vm();
    test_vm.registers[0] = 1024;
    test_vm.program = vec![17, 0, 0, 0];
    test_vm.run_once();
    assert_eq!(test_vm.heap.len(), 1024);
}
```

## 结束

耶，我们的虚拟机现在可以分配内存了！如果这对你来说很困惑，不用担心。现代内存管理非常复杂，需要一些实践来内化这些概念。在下一部分中，我们将为我们的汇编器添加字符串和打印功能。

### 原文链接及作者信息

- 原文链接：[Memory - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-11/)
- 作者名称：Fletcher

### 成语及典故

- 成语 & 典故：`A Faustian Bargain`: `浮士德式的交易 (A Faustian Bargain)`，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

### 专有名词及注释

- 汇编语言（Assembly Language）：一种低级编程语言，用于编写机器语言指令，通常用于硬件级编程。
- 堆（Heap）：用于存储在编译时无法预知空间需求的数据结构。
- 垃圾回收（Garbage Collection）：一种内存管理技术，用于自动回收不再使用的数据结构所占用的内存。
- 引用计数（Reference Counting）：一种垃圾收集机制，通过跟踪引用数量来决定何时释放内存。
- 标记 - 清扫（Mark and Sweep）：一种垃圾收集算法，通过标记可达对象和清扫未标记对象来回收内存。
- 操作系统（Operating System, OS）：管理和控制计算机硬件与软件资源的核心系统。
- 内存泄漏（Memory Leak）：由于疏忽或错误，程序未能释放已经不再使用的内存。
- 内存溢出 (Memory Overflow): 当程序尝试分配比可用内存更多的内存时，就会发生内存溢出。

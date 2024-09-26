# 20-性能基准测试 - 你想构建一个语言虚拟机吗？

- [20-性能基准测试 - 你想构建一个语言虚拟机吗？](#20-性能基准测试---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [性能基准测试](#性能基准测试)
        - [有用的基准测试](#有用的基准测试)
    - [添加基准 (Criterion)](#添加基准-criterion)
        - [Cargo](#cargo)
        - [基准测试 (Benches)](#基准测试-benches)
            - [缺失的函数](#缺失的函数)
        - [运行基准测试](#运行基准测试)
    - [保留基准测试图表](#保留基准测试图表)
    - [结束](#结束)
        - [俚语\&典故](#俚语典故)
        - [专有名词及注释](#专有名词及注释)

## 引言

我们玩得如此开心，以至于我们还没有编写任何性能基准测试！虽然编写它们并不是最令人兴奋的事情，但它们是重要的。

### 性能基准测试

关于基准测试，有两件事情需要理解：

1. 没有上下文 (`context`) 的单个基准测试是没有帮助的。

2. 当它们随时间跟踪时最有用。

你可以对任何东西进行基准测试，你可以调整它以在那个特定的基准测试中表现良好。

### 有用的基准测试

我发现在以下情况下基准测试最有用：

1. 当你从大型系统或应用程序的许多子组件中收集它们时。

2. 你保留历史记录，这样你就可以看到趋势 (`trends`)。

如果我们正在构建一个由多个子组件组成的系统，例如：

- Web 前端

- 后端

- 数据库

- 负载均衡器 (`Load Balancers`)

对于这些有用的基准测试将是：

- 负载均衡器在服务器类型 X 上可以维持多少并发 (`concurrent`) 连接？我并不在乎它计算质数 (`prime numbers`) 的速度有多快。

- 数据库服务器的缓存层 (`caching layer`) 有多有效？我不在乎它的 RAM 时序快了 3 纳秒。

无论如何，废话少说。简而言之，我们应该孤立地对小组件进行基准测试，并跟踪它们的性能随时间的变化。

## 添加基准 (Criterion)

Rust 有一个基准测试系统，但它在稳定版中不可用。基准测试的 go-to crate 是 [Criterion](https://github.com/japaric/criterion.rs)，让我们开始吧。

### Cargo

在你的 `Cargo.toml` 中添加以下内容：

```toml
[[bench]]
name = "iridium"
harness = false
```

### 基准测试 (Benches)

接下来，在 `benches/` 目录下创建一个新目录。没错，和 `src/` 同级。这个文件是我们将放置我们的基准测试函数的地方。

在 `benches/` 中，创建 `iridium.rs` 并包含以下内容：

```rust
#[macro_use]
extern crate criterion;
extern crate iridium;

use criterion::Criterion;
use iridium::vm::VM;
use iridium::assembler::{PIE_HEADER_PREFIX, PIE_HEADER_LENGTH};
```

让我们从一个简单的基准测试开始，即 VM 执行算术指令。接下来在 iridium.rs 文件中添加：

```rust
mod arithmetic {
    use super::*;

    fn execute_add(c: &mut Criterion) {
        let clos = {
            let mut test_vm = get_test_vm();
            test_vm.program = vec![1, 0, 1, 2];
            test_vm.run_once();
        };

        c.bench_function(
            "execute_add",
            move |b| b.iter(|| clos)
        );
    }
}
```

如你所见，它几乎与我们的测试相同：

```rust
#[test]
fn test_add_opcode() {
    let mut test_vm = get_test_vm();
    test_vm.program = vec![1, 0, 1, 2];
    test_vm.program = prepend_header(test_vm.program);
    test_vm.run();
    assert_eq!(test_vm.registers[2], 15);
}
```

我不知道我们可以将一系列表达式绑定到一个变量！这太酷了。我们所做的就是复制每个算术指令的测试并将它们添加进去。

#### 缺失的函数

你可能注意到我们没有 `get_test_vm` 函数，也没有 `prepend_header` 函数。由于这些在 `vm.rs` 的测试之外变得有用，让我们将它们提升为公共函数。

前往 `src/vm.rs`。我们有点重新排列要做。

首先，让我们将 `prepend_header` 和 `get_test_vm` 函数提升到 `VM` trait 并使它们公共。这样，我们就可以像这样调用它们：`VM::prepend_header(…​)`。这将会破坏一些东西：

1. 将 `use assembler::PIE_HEADER_PREFIX;` 更改为 `use assembler::{PIE_HEADER_PREFIX, PIE_HEADER_LENGTH};`

2. 在每个使用 `prepend_header` 或 `get_test_vm` 的测试中，用 `VM::` 作为函数调用的前缀。我使用搜索和替换来做这个，但你可以使用仓库中的代码如果你更喜欢。 =)

现在回到 `benches/iridium.rs`，我们可以做：

```rust
mod arithmetic {
    use super::*;

    fn execute_add(c: &mut Criterion) {
        let clos = {
            let mut test_vm = VM::get_test_vm();
            test_vm.program = vec![1, 0, 1, 2];
            test_vm.run_once();
        };

        c.bench_function(
            "execute_add",
            move |b| b.iter(|| clos)
        );
    }
}
```

### 运行基准测试

快到了！现在我们需要使用 criterion 宏来设置我们的基准测试组。在算术 (`arithmetic`) 模块的末尾，放置：

```rust
criterion_group!{
    name = arithmetic;
    config = Criterion::default();
    targets = execute_add, execute_sub, execute_mul, execute_div,
}
```

是的，宏中的括号应该是 `{` 和 `}`，而不是 `(` 和 `)`。作为 `iridium.rs` 的最后一行，放置：

```rust
criterion_main!(arithmetic::arithmetic);
```

就是这样！添加基准测试函数（或查看仓库）为每个算术运算符。从 `iridium/` 目录的根目录，我们现在可以运行 `cargo bench`，我们应该看到：

```
<snip 我们不关心的很多东西>

运行 target/release/deps/iridium-b5264c6303e130cb

运行 0 个测试

测试结果：ok。0 通过；0 失败；0 忽略；0 测量；0 筛选

运行 target/release/deps/iridium-59eea43f05a081ed
execute_add             time:   [0.0000 ps 0.0000 ps 0.0000 ps]
                   change: [-35.123% +1503.3% +5337.3%] (p = 0.39 > 0.05)
                   No change in performance detected.
Found 13 outliers among 100 measurements (13.00%)
4 (4.00%) high mild
9 (9.00%) high severe

execute_sub             time:   [0.0000 ps 0.0000 ps 0.0000 ps]
                   change: [-54.712% -12.150% +78.688%] (p = 0.73 > 0.05)
                   No change in performance detected.
Found 13 outliers among 100 measurements (13.00%)
5 (5.00%) high mild
8 (8.00%) high severe

execute_mul             time:   [0.0000 ps 0.0000 ps 0.0000 ps]
                   change: [-50.926% -1.5089% +101.21%] (p = 0.97 > 0.05)
                   No change in performance detected.
Found 12 outliers among 100 measurements (12.00%)
4 (4.00%) high mild
8 (8.00%) high severe

execute_div             time:   [0.0000 ps 0.0000 ps 0.0000 ps]
                   change: [-48.559% -5.5472% +73.134%] (p = 0.87 > 0.05)
                   No change in performance detected.
Found 11 outliers among 100 measurements (11.00%)
3 (3.00%) high mild
8 (8.00%) high severe
```

如果你查看 `targets/criterion/` 目录，你将看到 criterion 输出的漂亮图表。

## 保留基准测试图表

如果我们想保留这些图表 (`Graphs`)，以便我们可以随着时间比较运行结果，我们需要将它们放在某个地方。由于我们使用 Appveyor、Gitlab 和 Travis 为多个平台构建 Iridium，每个平台都将运行基准测试，我们需要保留它们。最简单的方法是指定它们是工件，并保留每个构建的图表，以及每个平台的二进制文件。

我不会在这里详细介绍，但你可以查看 `.travis.yml`、`.gitlab-ci.yml` 和 `appveyor.yml` 来看我是如何做到的。

## 结束

我认为这是一个很好的停止点 (`stopping point`)。我们需要编写更多的基准测试，我们将在进行中添加更多。

下次见！

原文出处：[Benchmarks - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-20/)
作者名：Fletcher

### 俚语&典故

1. `A Faustian Bargain`：浮士德式的交易 (A Faustian Bargain)，这个概念源于德国关于浮士德的传说，浮士德为了追求知识和权力，与魔鬼签订了契约，以自己的灵魂作为交换。此外，这个词也可以用来形容人们为了获取某项服务或产品，可能需要牺牲一定的隐私权或其他权利的情形。

2. `Zergling`: 是来自于游戏《星际争霸》（StarCraft）中的一个单位，属于 Zerg 种族的基本战斗单位。在这里，作者可能是以幽默的方式，将线程比作是快速、大量生产的小型战斗单位，以此来形象地描述线程的创建和管理工作。在翻译时，可以保留原文`zergling`，因为它已经成为了游戏文化中的一个专有名词，或者可以解释性地翻译为`快速生成的战斗单位`或`轻量级线程`。

### 专有名词及注释

1. RAM 是随机存取存储器（Random Access Memory）的缩写，它是一种计算机内存，用于临时存储操作系统、应用程序和数据，以便快速访问和处理。RAM 是计算机硬件的关键组成部分，它决定了计算机能够同时运行多少程序以及这些程序的运行速度。
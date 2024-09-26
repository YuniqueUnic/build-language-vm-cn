# 02-RUST Virtual Machine Learning

## 今日学习总结

> 当前北京时间：2024 年 09 月 13 日 15:51:24

### 进度

- [x] build_vm_part_16.md

### 笔记

**Clap** 是一个用于命令行参数解析的 Rust 库，它提供了一种简单的方式来解析命令行参数，并生成帮助信息。

- [derive](https://docs.rs/clap/latest/clap/_derive/index.html#mixing-builder-and-derive-apis) 模式：允许使用 derive 模式来定义命令行参数，同时使用 builder 模式来定义更复杂的参数组合。
- [builder](https://docs.rs/clap/latest/clap/_tutorial/chapter_0/index.html) 模式：允许使用 builder 模式来定义命令行参数，并使用 derive 模式来定义更复杂的参数组合。

clap 的 [FAQ](https://docs.rs/clap/latest/clap/_faq/index.html#)

1. [Comparisons](https://docs.rs/clap/latest/clap/_faq/index.html#comparisons)
    1. [How does clap compare to structopt?](https://docs.rs/clap/latest/clap/_faq/index.html#how-does-clap-compare-to-structopt)
    2. [What are some reasons to use clap? (The Pitch)](https://docs.rs/clap/latest/clap/_faq/index.html#what-are-some-reasons-to-use-clap-the-pitch)
    3. [What are some reasons not to use clap? (The  Anti Pitch)](https://docs.rs/clap/latest/clap/_faq/index.html#what-are-some-reasons-not-to-use-clap-the-anti-pitch)
    4. [Reasons to use clap](https://docs.rs/clap/latest/clap/_faq/index.html#reasons-to-use-clap)
2. [How many approaches are there to create a parser?](https://docs.rs/clap/latest/clap/_faq/index.html#how-many-approaches-are-there-to-create-a-parser)
3. [When should I use the builder vs derive APIs?](https://docs.rs/clap/latest/clap/_faq/index.html#when-should-i-use-the-builder-vs-derive-apis)
4. [Why is there a default subcommand of help?](https://docs.rs/clap/latest/clap/_faq/index.html#why-is-there-a-default-subcommand-of-help)

**clap 优秀教程：**

1. Derive/Builder: 十分优秀的教程：[深入探索 Rust 的 clap 库：命令行解析的艺术](https://blog.csdn.net/Hedon954/article/details/139578613)
2. Derive/Builder: [Rust 命令行库 Clap 快速入门教程](https://youerning.top/post/rust/rust-clap-tutorial/)
3. Derive/Builder: [Writing a CLI Tool in Rust with Clap](https://www.shuttle.rs/blog/2023/12/08/clap-rust)
4. Builder: [Rust: Take your CLI to the Next Level with Clap](https://medium.com/@itsuki.enjoy/rust-take-your-cli-to-the-next-level-with-clap-a0f05875ef45)

今日有些疲惫，因而代码并没有写什么，更多的是看资料和了解要编写的 Clap 内容。后续将 clap 部分加入到 lrvm 中。
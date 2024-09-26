# 30-清理时间 - 你想构建一个语言虚拟机吗？

- [30-清理时间 - 你想构建一个语言虚拟机吗？](#30-清理时间---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [范围](#范围)
    - [Rustfmt](#rustfmt)
    - [现在……让我们看看 Clippy](#现在让我们看看-clippy)
    - [结束](#结束)
    - [专有名词及注释](#专有名词及注释)

## 引言

虽然处理集群化工作非常有趣，但我们积累 (`accumulated`) 了相当多的技术债务 (`technical debt`)。有大量的警告，我甚至不敢运行 clippy。因此，这篇文章全是关于逐一清理这些警告的。它可能不会像添加功能那样令人兴奋，但是确保花时间进行清理同样重要，甚至更为重要。技术债务的增长速度有时比信用卡 (`credit card`) 债务还要快。

## 范围

让我们通过在`master`分支上的最新提交运行`cargo check`来看看我们面临的问题。

```sh
Checking iridium v0.0.29 (file:///Users/fletcher/Projects/subnetzero/iridium)
warning: unused import: `std::fmt`
--> src/assembler/mod.rs:12:5
|
12 | use std::fmt;
|     ^^^^^^^^
|
= note: #[warn(unused_imports)] on by default

warning: unused import: `log`
--> src/assembler/mod.rs:16:5
|
16 | use log;
|     ^^^

warning: unused imports: `Arc`, `RwLock`
--> src/repl/mod.rs:9:17
|
9 | use std::sync::{RwLock, Arc, mpsc};
|                 ^^^^^^  ^^^

warning: unused import: `cluster::manager::Manager`
--> src/repl/mod.rs:18:5
|
18 | use cluster::manager::Manager;
|     ^^^^^^^^^^^^^^^^^^^^^^^^^

warning: unused import: `Assembler`
--> src/vm.rs:12:55
|
12 | use assembler::{PIE_HEADER_LENGTH, PIE_HEADER_PREFIX, Assembler};
|                                                       ^^^^^^^^^

warning: unused import: `RwLock`
--> src/cluster/client.rs:5:22
|
5 | use std::sync::{Arc, RwLock, Mutex};
|                      ^^^^^^

warning: unused import: `cluster::message::IridiumMessage`
--> src/cluster/client.rs:9:5
|
9 | use cluster::message::IridiumMessage;
|     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

warning: unused import: `cluster::manager::Manager`
--> src/cluster/client.rs:10:5
|
10 | use cluster::manager::Manager;
|     ^^^^^^^^^^^^^^^^^^^^^^^^^

warning: unused imports: `Arc`, `RwLock`
--> src/cluster/mod.rs:6:17
|
6 | use std::sync::{Arc, RwLock};
|                 ^^^  ^^^^^^

warning: unused variable: `bytes_read`
--> src/cluster/server.rs:18:17
|
18 |             let bytes_read = client.reader.read(&mut buf);
|                 ^^^^^^^^^^ help: consider using `_bytes_read` instead
|
= note: #[warn(unused_variables)] on by default

warning: unused variable: `connection_manager`
--> src/cluster/server.rs:9:33
|
9 | pub fn listen(addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
|                                 ^^^^^^^^^^^^^^^^^^ help: consider using `_connection_manager` instead

warning: unused variable: `args`
--> src/repl/mod.rs:342:35
|
342 |     fn cluster_members(&mut self, args: &[&str]) {
|                                   ^^^^ help: consider using `_args` instead

warning: unused variable: `offset`
--> src/vm.rs:458:21
|
458 |                 let offset = self.registers[self.next_8_bits() as usize] as usize;
|                     ^^^^^^ help: consider using `_offset` instead

warning: field is never used: `tx`
--> src/cluster/client.rs:17:5
|
17 |     tx: Option<Arc<Mutex<Sender<String>>>>,
|     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
|
= note: #[warn(dead_code)] on by default

warning: method is never used: `w`
--> src/cluster/client.rs:44:5
|
44 |     fn w(&mut self, msg: &str) -> bool {
|     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

warning: type alias is never used: `NodeCollection`
--> src/cluster/mod.rs:11:1
|
11 | type NodeCollection = HashMap<NodeAlias, ClusterClient>;
| ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

warning: unused `std::result::Result` which must be used
--> src/vm.rs:461:17
|
461 |                 buf.as_mut().write_i32::<LittleEndian>(data);
|                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
|
= note: #[warn(unused_must_use)] on by default
= note: this `Result` may be an `Err` variant, which should be handled

warning: unused `std::result::Result` which must be used
--> src/vm.rs:466:17
|
466 |                 buf.as_mut().write_i32::<LittleEndian>(data);
|                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
|
= note: this `Result` may be an `Err` variant, which should be handled

warning: unused `std::result::Result` which must be used
--> src/cluster/client.rs:41:9
|
41 |         self.raw_stream.write(&alias.as_bytes());
|         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
|
= note: this `Result` may be an `Err` variant, which should be handled

warning: unused variable: `events`
--> src/bin/iridium.rs:75:25
|
75 |                     let events = vm.run();
|                         ^^^^^^ help: consider using `_events` instead
|
= note: #[warn(unused_variables)] on by default

Finished dev [unoptimized + debuginfo] target(s) in 2.41s
```

所以……是的。情况并不太差。让我们看看`cargo fmt`是否能解决其中一些问题。

## Rustfmt

我们有一个最小的.rustfmt.toml，但让我们生成一个默认的配置文件，我们可以使用：`rustfmt --print-config default .rustfmt.toml`。如果你查看里面，你会看到更多的规则。我不得不做一些调整，所以你将需要替换它为：

```toml
max_width = 100
hard_tabs = false
tab_spaces = 4
newline_style = "Auto"
use_small_heuristics = "Default"
indent_style = "Block"
wrap_comments = false
comment_width = 80
normalize_comments = false
format_strings = false
format_macro_matchers = false
format_macro_bodies = true
empty_item_single_line = true
struct_lit_single_line = true
fn_single_line = false
where_single_line = false
imports_indent = "Block"
imports_layout = "Mixed"
merge_imports = false
reorder_imports = true
reorder_modules = true
reorder_impl_items = false
type_punctuation_density = "Wide"
space_before_colon = false
space_after_colon = true
spaces_around_ranges = false
binop_separator = "Front"
remove_nested_parens = true
combine_control_expr = true
struct_field_align_threshold = 0
match_arm_blocks = true
force_multiline_blocks = false
fn_args_density = "Tall"
brace_style = "SameLineWhere"
control_brace_style = "AlwaysSameLine"
trailing_semicolon = true
trailing_comma = "Vertical"
match_block_trailing_comma = false
blank_lines_upper_bound = 1
blank_lines_lower_bound = 0
edition = "2018"
merge_derives = true
use_try_shorthand = false
use_field_init_shorthand = false
force_explicit_abi = true
condense_wildcard_suffixes = false
color = "Auto"
required_version = "0.99.5"
unstable_features = false
disable_all_formatting = false
skip_children = false
hide_parse_errors = false
error_on_line_overflow = false
error_on_unformatted = false
report_todo = "Never"
report_fixme = "Never"
ignore = []
emit_mode = "Files"
make_backup = false
```

确保你已经运行：

```sh
rustup toolchain install nightly
rustup component add rustfmt-preview --toolchain=nightly
```

然后实际进行格式化，运行：

现在如果我们运行`cargo check`……我们仍然看到很多未使用的导入。这些应该很容易清理！我不会在这篇文章中详细说明。=）

<时间流逝> (`<time passes>`)

所有都完成了！如果你对完整的代码感兴趣，请看这里：<https://gitlab.com/subnetzero/iridium/commit/eee41a00b56351fc0310e77abcb48dfd3a125ef7>

## 现在……让我们看看 Clippy

如果我们运行：`cargo +nightly clippy`，然后我们得到：

```sh
warning: redundant field names in struct initialization
  --> src/repl/mod.rs:43:13
   |
43 |             vm: vm,
   |             ^^^^^^ help: replace it with: `vm`
   |
   = note: #[warn(clippy::redundant_field_names)] on by default
   = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#redundant_field_names

warning: returning the result of a let binding from a block. Consider returning the expression directly.
   --> src/vm.rs:557:9
    |
557 |         starting_offset
    |         ^^^^^^^^^^^^^^^
    |
    = note: #[warn(clippy::let_and_return)] on by default
note: this expression can be directly returned
   --> src/vm.rs:556:31
    |
556 |         let starting_offset = rdr.read_u32::<LittleEndian>().unwrap() as usize;
    |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#let_and_return

warning: using `clone` on a `Copy` type
   --> src/assembler/mod.rs:358:46
    |
358 |                 *starting_instruction = Some(self.current_instruction.clone())
    |                                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: try removing the `clone` call: `self.current_instruction`
    |
    = note: #[warn(clippy::clone_on_copy)] on by default
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#clone_on_copy

warning: using `clone` on a `Copy` type
   --> src/assembler/mod.rs:364:46
    |
364 |                 *starting_instruction = Some(self.current_instruction.clone())
    |                                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: try removing the `clone` call: `self.current_instruction`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#clone_on_copy

warning: redundant pattern matching, consider using `is_ok()`
  --> src/cluster/client.rs:38:16
   |
38 |         if let Ok(_) = self.raw_stream.write(&alias.as_bytes()) {
   |         -------^^^^^------------------------------------------- help: try this: `if self.raw_stream.write(&alias.as_bytes()).is_ok()`
   |
   = note: #[warn(clippy::if_let_redundant_pattern_matching)] on by default
   = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#if_let_redundant_pattern_matching

warning: this argument is passed by value, but not consumed in the function body
  --> src/cluster/manager.rs:28:41
   |
28 |     pub fn del_client(&mut self, alias: NodeAlias) {
   |                                         ^^^^^^^^^ help: consider changing the type to: `&str`
   |
   = note: #[warn(clippy::needless_pass_by_value)] on by default
   = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#needless_pass_by_value

warning: you seem to want to iterate on a map's keys
  --> src/cluster/manager.rs:34:27
   |
34 |         for (alias, _) in &self.clients {
   |                           ^^^^^^^^^^^^^
   |
   = note: #[warn(clippy::for_kv_map)] on by default
   = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#for_kv_map
help: use the corresponding method
   |
34 |         for alias in self.clients.keys() {
   |             ^^^^^    ^^^^^^^^^^^^^^^^^^^

warning: this argument is passed by value, but not consumed in the function body
 --> src/cluster/server.rs:9:54
  |
9 | pub fn listen(addr: SocketAddr, _connection_manager: Arc<RwLock<Manager>>) {
  |                                                      ^^^^^^^^^^^^^^^^^^^^ help: consider taking a reference instead: `&Arc<RwLock<Manager>>`
  |
  = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#needless_pass_by_value

warning: useless use of `format!`
   --> src/repl/mod.rs:319:27
    |
319 |         self.send_message(format!("Started cluster server!"));
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: consider using .to_string(): `"Started cluster server!".to_string()`
    |
    = note: #[warn(clippy::useless_format)] on by default
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#useless_format

warning: useless use of `format!`
   --> src/repl/mod.rs:324:27
    |
324 |         self.send_message(format!("Attempting to join cluster..."));
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: consider using .to_string(): `"Attempting to join cluster...".to_string()`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#useless_format

warning: useless use of `format!`
   --> src/repl/mod.rs:329:31
    |
329 |             self.send_message(format!("Connected to cluster!"));
    |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: consider using .to_string(): `"Connected to cluster!".to_string()`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#useless_format

warning: useless use of `format!`
   --> src/repl/mod.rs:338:31
    |
338 |             self.send_message(format!("Could not connect to cluster!"));
    |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: consider using .to_string(): `"Could not connect to cluster!".to_string()`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#useless_format

warning: useless use of `format!`
   --> src/repl/mod.rs:343:27
    |
343 |         self.send_message(format!("Listing Known Nodes:"));
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ help: consider using .to_string(): `"Listing Known Nodes:".to_string()`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#useless_format

warning: you don't need to add `&` to both the expression and the patterns
  --> src/vm.rs:27:9
   |
27 | /         match &self {
28 | |             &VMEventType::Start => 0,
29 | |             &VMEventType::GracefulStop { code } => *code,
30 | |             &VMEventType::Crash { code } => *code,
31 | |         }
   | |_________^
   |
   = note: #[warn(clippy::match_ref_pats)] on by default
   = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#match_ref_pats
help: try
   |
27 |         match self {
28 |             VMEventType::Start => 0,
29 |             VMEventType::GracefulStop { code } => *code,
30 |             VMEventType::Crash { code } => *code,
   |

warning: casting u8 to i32 may become silently lossy if types change
   --> src/vm.rs:423:27
    |
423 |                 let uv1 = self.next_8_bits() as i32;
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^ help: try: `i32::from(self.next_8_bits())`
    |
    = note: #[warn(clippy::cast_lossless)] on by default
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#cast_lossless

warning: casting u8 to i32 may become silently lossy if types change
   --> src/vm.rs:424:27
    |
424 |                 let uv2 = self.next_8_bits() as i32;
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^ help: try: `i32::from(self.next_8_bits())`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#cast_lossless

warning: redundant pattern matching, consider using `is_ok()`
   --> src/vm.rs:464:24
    |
464 |                 if let Ok(_) = buf.as_mut().write_i32::<LittleEndian>(data) {
    |                 -------^^^^^----------------------------------------------- help: try this: `if buf.as_mut().write_i32::<LittleEndian>(data).is_ok()`
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#if_let_redundant_pattern_matching

warning: the variable `c` is used as a loop counter. Consider using `for (c, item) in self.stack.drain(new_len..).enumerate()` or similar iterators
   --> src/vm.rs:477:40
    |
477 |                 for removed_element in self.stack.drain(new_len..) {
    |                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = note: #[warn(clippy::explicit_counter_loop)] on by default
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#explicit_counter_loop

warning: the variable `c` is used as a loop counter. Consider using `for (c, item) in self.stack.drain(new_len..).enumerate()` or similar iterators
   --> src/vm.rs:495:40
    |
495 |                 for removed_element in self.stack.drain(new_len..) {
    |                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#explicit_counter_loop

    Finished dev [unoptimized + debuginfo] target(s) in 12.96s
```

呼，好的。它们大部分是风格上/习惯性 (`stylistic/idiomatic`) 的问题。然而，这个警告是一个很好的发现：

```sh
warning: casting u8 to i32 may become silently lossy if types change
   --> src/vm.rs:423:27
    |
423 |                 let uv1 = self.next_8_bits() as i32;
    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^ help: try: `i32::from(self.next_8_bits())`
    |
    = note: #[warn(clippy::cast_lossless)] on by default
    = help: for further information visit https://rust-lang-nursery.github.io/rust-clippy/v0.0.212/index.html#cast_lossless
```

## 结束

我在 w 函数中允许存在未使用的代码，因为我们稍后将会使用它，以及其中一个测试函数。如果您想查看经过 Clippy 审核的版本，请查看：<https://gitlab.com/subnetzero/iridium/commit/6206685d402651f0dd0f98b034011fef3990bff6>

直到第 31 部分！

**原文出处**：[https://blog.subnetzero.io/post/building-language-vm-part-30/](https://blog.subnetzero.io/post/building-language-vm-part-30/)  
**作者名**：Fletcher

## 专有名词及注释

- `rustfmt` 是一个 Rust 语言的代码格式化工具，它能够自动地重新格式化 Rust 代码，以保持代码风格的一致性。`rustfmt` 的目的是减少代码风格上的争论，并使得代码库中的所有代码看起来像是同一个人编写的。它是一个命令行工具，可以单独运行，也可以集成到编辑器或 IDE 中。
- `Clippy` 是一个 Rust 语言的静态分析工具，它专注于发现常见的编程错误、性能问题和代码风格问题。`Clippy` 提供了一系列的检查器（linter），这些检查器会分析 Rust 代码并提出改进建议。`Clippy` 的目标是帮助开发者写出更清晰、更安全、更高效的 Rust 代码。
- `Clippy` 通常与 `rustfmt` 结合使用，以确保代码不仅在格式上符合标准，而且在内容上也遵循最佳实践。使用这两个工具可以帮助提高代码质量，并减少潜在的问题。

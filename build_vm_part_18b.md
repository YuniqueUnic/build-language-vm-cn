# 18b-REPL 命令解析 - 你想构建一个语言虚拟机吗？

- [18b-REPL 命令解析 - 你想构建一个语言虚拟机吗？](#18b-repl-命令解析---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [命令等等 (Commands and Such)](#命令等等-commands-and-such)
        - [步骤 1：命令解析器结构](#步骤-1命令解析器结构)
        - [步骤 2：拆分函数](#步骤-2拆分函数)
        - [步骤 3：运行函数更改](#步骤-3运行函数更改)
            - [UTF-8 烦恼 (Annoyances)](#utf-8-烦恼-annoyances)
    - [结束](#结束)

## 引言

你好！在這一部分中，我们将要在 REPL 中去解析用户输入的命令。目前，还不够灵活；用戶不能输入这样的：

```sh
.load_file /path/to/file
```

目前，他们必须像这样才行：

```sh
.load_file\r
Please enter file path: /path/to/file\r
```

## 命令等等 (Commands and Such)

首先，我们需要做一些小改动。你可能已经注意到，我们在汇编语言中使用了 `.` 字符，所以我们需要使用其他字符。目前，我们将使用 `!` 来代替。

让我们将这个功能作为 `repl` 模块的一个子模块。你可以创建 `src/repl/command_parser`。

### 步骤 1：命令解析器结构

这并不复杂。我们只需要一个函数来对输入进行标记化：

```rust
pub struct CommandParser {}

impl CommandParser {
    pub fn tokenize(input: &str) -> Vec<&str> {
        let split = input.split_whitespace();
        let vec: Vec<&str> = split.collect();
        vec
    }
}
```

这里没有必要创建这个结构体的实例。我们只需要按空格 (`whitespace`) 分割用户输入的字符串，并返回这些标记。

别忘了在 `src/repl/mod.rs` 中添加 `pub mod command_parser;`。

### 步骤 2：拆分函数

你知道我们有一个大的匹配块来检查 `.whatever` 吗？每一个选项都应该是它自己的函数，像这样：

```rust
fn quit(&mut self, args: &[&str]) {
    println!("Farewell! Have a great day!");
    std::process::exit(0);
}
```

我不会在本教程中展示每一个函数，因为它只是简单的复制粘贴。

### 步骤 3：运行函数更改

更有趣的是我们必须如何改变解析逻辑。这是 `src/repl/mod.rs` 中新的 `run` 函数：

```rust
pub fn run(&mut self) {
    println!("Welcome to Iridium! Let's be productive!");
    loop {
        // 这会在每次循环中分配一个新的字符串，用来存储用户输入的内容。
        // TODO: 想办法在循环外分配这个，并在每次循环中重复使用它
        let mut buffer = String::new();

        // 阻塞调用，直到用户输入命令
        let stdin = io::stdin();

        // 令人烦恼的是，`print!` 不像 `println!` 那样自动刷新 stdout，所以我们
        // 必须在那里做，以便用户能看到我们的 `>>>` 提示。
        print!(">>> ");
        io::stdout().flush().expect("Unable to flush stdout");

        // 在这里我们将查看用户给我们的字符串。
        stdin
            .read_line(&mut buffer)
            .expect("Unable to read line from user");

        let historical_copy = buffer.clone();
        self.command_buffer.push(historical_copy);

        if buffer.starts_with("!") {
            self.execute_command(&buffer);
        } else {
            let program = match program(CompleteStr(&buffer)) {
                Ok((_remainder, program)) => {
                    program
                },
                Err(e) => {
                    println!("Unable to parse input: {:?}", e);
                    continue;
                }
            };
            self.vm.program.append(&mut program.to_bytes(&self.asm.symbols));
            self.vm.run_once();
        }
    }
}
```

这里是新的 `execute_command` 函数：

```rust
fn execute_command(&mut self, input: &str) {
    let args = CommandParser::tokenize(input);
    match args[0] {
        "!quit" => self.quit(&args[1..]),
        "!history" => self.history(&args[1..]),
        "!program" => self.program(&args[1..]),
        "!clear_program" => self.clear_program(&args[1..]),
        "!clear_registers" => self.clear_registers(&args[1..]),
        "!registers" => self.registers(&args[1..]),
        "!symbols" => self.symbols(&args[1..]),
        "!load_file" => self.load_file(&args[1..]),
        "!spawn" => self.spawn(&args[1..]),
        _ => { println!("Invalid command") }
    };
}
```

注意我们如何从传递给每个独立函数的切片中剥离出命令的部分，这样它们就得到了参数而不是带有`!命令`的副本。

#### UTF-8 烦恼 (Annoyances)

因为 Rust 中的所有字符串都是 UTF-8 编码，你不能像期望的那样用 `&buffer[0]` 来检查第一个字符。我曾对如何不制作大量副本就检查第一个字符是否为 `"!"` 感到烦恼。

幸运的是，我发现了 `starts_with` 函数！似乎有很多有用的便利函数潜伏着。

## 结束

本文到此结束！这些更改将包含在 0.0.18 版本中。我正在努力找出一种好的方法来同步版本与教程。代码可以在 [GitLab](https://gitlab.com/subnetzero/iridium/tags/0.0.18b) 上找到！

原文出处：[REPL Command Parsing - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-18b/)
作者名：Fletcher

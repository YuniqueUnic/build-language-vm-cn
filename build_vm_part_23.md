# 23-SSH 服务器：第一部分 - 你想构建一个语言虚拟机吗？

- [23-SSH 服务器：第一部分 - 你想构建一个语言虚拟机吗？](#23-ssh-服务器第一部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [多用户访问](#多用户访问)
        - [权限](#权限)
    - [安全远程访问](#安全远程访问)
        - [Thrussh](#thrussh)
    - [实现](#实现)
        - [第 1 步：启用 SSH CLI 标志](#第-1-步启用-ssh-cli-标志)
            - [添加 Rust 代码](#添加-rust-代码)
            - [SSH 服务器实现](#ssh-服务器实现)
            - [启动 SSH 服务器](#启动-ssh-服务器)
    - [结束](#结束)
        - [专有名词及注释](#专有名词及注释)

## 引言

目前，我们有两种方式与 Iridium 虚拟机交互：

1. 运行 `iridium <somefile>`

2. REPL
    1. 但 Iridium 虚拟机不仅仅是一个语言解释器；它应该是一个平台，你可以在其中部署应用程序、检查它们的状态、故障转移 (`failover`) 等等。这意味着我们需要一种方法，以便：

3. 多个人与运行中的虚拟机交互

4. 安全地远程 (`remotely`) 访问虚拟机

## 多用户访问

实现这一点并不太难。我们可以允许多个人通过 Unix 管道连接，但这将限制只能从该服务器访问。更好的解决方案是绑定到网络接口并监听网络连接。

### 权限

更复杂的是设计一个权限 (`Permissions`) 系统。如果我运行了两个应用程序，我可能希望给一个同事访问其中一个的权限，但不包括另一个。我们如何定义用户？我们希望访问控制有多细粒度 (`granular`)？我们应该与 Active Directory/LDAP 集成吗？

都是好问题！现在，我们将采用全有或全无的模型。如果你可以通过 SSH 连接到虚拟机，你可以做任何事情。

## 安全远程访问

开启一个端口并开始接受流量是很简单 (`trivial`) 的。我们可以在 `\n` 处断开，解析命令等。除了一个细节……所有这些都会发生在明文传输中；你输入并发送到虚拟机的任何内容都可能被你和虚拟机之间的某些东西读取。

为了解决这个问题，我们可以使用 SSH。这是许多开发者已经在使用的，它将使与 Iridium 虚拟机的交互类似于与通用 Linux 服务器的交互。

| | |
|---|---|
| 重要 | 不要尝试实现你自己的加密解决方案！我一次又一次地看到新编码者这样做。请使用一个经过审查 (`examined`) 和验证 (`vetted`) 的现有库。|

### Thrussh

我们将使用一个名为 `thrussh` 的 crate 为 Iridium 添加一个 SSH 服务器。你可以在这里找到它：<https://docs.rs/thrussh/0.20.0/thrussh/> 。将它添加到你的依赖项中。我们还需要 `thrussh_keys`，`futures`，和 `tokio`。要添加到 `Cargo.toml` 中的行是：

```toml
thrussh = "0.20.0"
thrussh-keys = "0.9.5"
futures = "0.1.24"
tokio = "0.1.8"
```

| | |
|---|---|
| 重要 | 它依赖于 libsodium。这意味着你需要安装它。在 OSX 上，它是 `brew install libsodium`。|

你还需要在 lib.rs 中为它们添加 `externs`：

```rust
extern crate thrussh;
extern crate thrussh_keys;
extern crate futures;
extern crate tokio;
```

## 实现

这应该相当简单。我们需要做以下几件事：

> 1. Add CLI flag for turning on the SSH server feature
> 2. Add CLI flag to specify the port it will listen on
> 3. Add a REPL command to add an authorized key
> 4. Add an iridium sub-command to add an authorized key
> 5. Create an SSH server in a background thread
> 6. Give each connection a REPL

1. 为启用 SSH 服务器功能添加 CLI 标志
2. 添加 CLI 标志以指定它将监听的端口
3. 添加一个 REPL 命令以添加授权密钥
4. 添加一个 `iridium` 子命令以添加授权密钥
5. 在后台线程中创建一个 SSH 服务器
6. 为每个连接提供一个 REPL

……好吧，也许那并不简单。但让我们尝试一下！

### 第 1 步：启用 SSH CLI 标志

我们将在 `src/bin/cli.yml` 和 `src/bin/iridium.rs` 中完成前三件事。打开 `cli.yml` 并让我们添加标志以启用 SSH 功能：

```yaml
- ENABLE_SSH:
    help: Enables the SSH server component of Iridium VM
    required: false
    takes_value: false
    long: enable-ssh
```

现在如果我们编译（`make dev`）并运行 `iridium --help`，我们应该会看到：

```sh
$ iridium --help
iridium 0.0.23
Fletcher Haynes <fletcher@subnetzero.io>
Interpreter for the Iridium language

USAGE:
    iridium [FLAGS] [OPTIONS] [INPUT_FILE]

FLAGS:
        --enable-ssh    Enables the SSH server component of Iridium VM
    -h, --help          Prints help information
    -V, --version       Prints version information

OPTIONS:
    -t, --threads <THREADS>    Number of OS threads the VM will utilize

ARGS:
    <INPUT_FILE>    Path to the .iasm or .ir file to run
```

现在来设置一个端口……

```yaml
- SSH_PORT:
    help: Iridium 应该监听 SSH 连接的端口
    required: false
    takes_value: true
    long: ssh-port
    short: p
```

现在对于子命令。运行它的一个示例是：

```sh
iridium add-ssh-key /path/to/public/key
```

将它添加到 `cli.yml` 文件中：

```yaml
子命令:
    - add-ssh-key:
        about: 将公钥 (public key) 添加到授权访问此 VM 的密钥列表 (the list of keys authorized) 中
        version: "0.0.1"
        author: Fletcher Haynes <fletcher@subnetzero.io>
        参数:
            - PUB_KEY_FILE:
                help: 包含公钥的文件路径
                index: 1
                required: true
```

#### 添加 Rust 代码

前往 `src/bin/iridium.rs`。这是我们将放置一些代码来处理一些标志的地方。在 `main` 函数中，添加这个简单的 if 检查：

```rust
let yaml = load_yaml!("cli.yml");
let matches = App::from_yaml(yaml).get_matches();

if matches.is_present("add-ssh-key") {
    println!("用户尝试添加 SSH 密钥！");
    // 你可能需要在顶部添加 `use std;`
    std::process::exit(0);
}

```

这将告诉我们子命令 (`sub-command`) 是否被使用。现在，我们将打印一条消息。如果我们编译并测试，我们应该会看到：

```sh
$ iridium add-ssh-key test
用户尝试添加 SSH 密钥！
```

接下来，让我们检查是否传递了启用 SSH 标志。在检查 `add-ssh-key` 的下方添加这个：

```rust
if matches.is_present("enable-ssh") {
    println!("用户想要启用 SSH！");
    if matches.is_present("ssh-port") {
        println!("他们想使用端口 {:#?}", matches.value_of("ssh-port"));
    }
}

```

现在如果我们编译并运行它：

```sh
$ iridium --enable-ssh -p 2333
用户想要启用 SSH！
他们想使用端口 None
欢迎使用 Iridium! 让我们提高生产力！
>>> !quit
再见(Farewell)！祝你有美好的一天！
```

我们现在有参数的占位符 (`stubs`) 了。

#### SSH 服务器实现

现在我们需要制作一个 SSH 服务器。这将是一个重要的、独立的 Iridium 子组件，所以让我们把它放在自己的模块中：`src/ssh.rs`。

打开那个文件，并在其中放置：

```rust
use std;

use futures;
use thrussh_keys;
use std::sync::Arc;
use thrussh::*;
use thrussh::server::{Auth, Session};
use thrussh_keys::*;
```

一如既往，我们将从一个基本结构开始。=)

```rust
#[derive(Clone, Debug)]
pub struct Server {

}
```

现在，我们粘贴一大块 (`blob`) futures 代码。我应该指出，我不喜欢 futures 编码模型。就像很多 😠

但我们稍后会详细修改这部分。

```rust
impl server::Server for Server {
   type Handler = Self;
   fn new(&self) -> Self {
       self.clone()
   }
}

impl server::Handler for Server {
   type Error = std::io::Error;
   type FutureAuth = futures::Finished<(Self, server::Auth), Self::Error>;
   type FutureUnit = futures::Finished<(Self, server::Session), Self::Error>;
   type FutureBool = futures::Finished<(Self, server::Session, bool), Self::Error>;

   fn finished_auth(self, auth: Auth) -> Self::FutureAuth {
       futures::finished((self, auth))
   }
   fn finished_bool(self, session: Session, b: bool) -> Self::FutureBool {
       futures::finished((self, session, b))
   }
   fn finished(self, session: Session) -> Self::FutureUnit {
       futures::finished((self, session))
   }

   fn auth_publickey(self, _: &str, _: &key::PublicKey) -> Self::FutureAuth {
       futures::finished((self, server::Auth::Accept))
   }
   fn data(self, channel: ChannelId, data: &[u8], mut session: server::Session) -> Self::FutureUnit {
       println!("data on channel {:?}: {:?}", channel, std::str::from_utf8(data));
       session.data(channel, None, data);
       futures::finished((self, session))
   }
}
```

这部分代码复制粘贴来自于 <https://docs.rs/thrussh/0.20.0/thrussh/>,并且我们可以修改它，同时我也推荐读一读他们的文档。

在我们开始修改服务器之前，让我们在后台线程中运行它，看看它是否能接受连接。

#### 启动 SSH 服务器

在 `bin/iridium.rs` 的末尾制作一个名为 `start_ssh_server` 的新函数，它有：

```rust
fn start_ssh_server()
 {
    let _t = std::thread::spawn(|| {
        let mut config = thrussh::server::Config::default();
        config.connection_timeout = Some(std::time::Duration::from_secs(600));
        config.auth_rejection_time = std::time::Duration::from_secs(3);
        let config = Arc::new(config);
        let sh = iridium::ssh::Server{};
        thrussh::server::run(config, "0.0.0.0:2223", sh);
    });
}
```

现在，回到 `main` 的开头，在检查 `ENABLE_SSH` 时，让我们调用 `start_ssh_server`：

```rust
if matches.is_present("ENABLE_SSH") {
    println!("用户想要启用 SSH！");
    if matches.is_present("SSH_PORT") {
        println!("他们想使用端口 {:#?}", matches.value_of("ssh-port"));
    }
    start_ssh_server()
}
```

现在让我们看看它是否甚至启动了一个服务器：

```sh
$ iridium --enable-ssh
用户想要启用 SSH！
欢迎使用 Iridium! 让我们提高生产力！
```

在另一个终端如果我们做：

```sh
$ ssh localhost -p 2223
ssh_exchange_identification: Connection closed by remote host
```

在我们的 Iridium REPL 中我们可以看到：

```sh
>>> err Error(Timer(TooLong))
```

哇哦！我们差不多完成了。

## 结束

这篇文章有点长，所以我们将在下一篇文章中继续。我们应该能够完成 SSH 访问。下次见！

原文出处：[SSH 服务器：第一部分 - 你想构建一个语言虚拟机吗？](https://blog.subnetzero.io/post/building-language-vm-part-23/)
作者名：Fletcher

### 专有名词及注释

- SSH 服务器 (Secure Shell Server): SSH 服务器（Secure Shell Server）是一种网络协议和相关的服务器软件，它允许用户通过加密的网络连接远程访问另一台计算机。SSH 提供了强大的认证、加密和完整性保护，以防止数据在传输过程中被窃听、篡改或伪造
- OSX (Mac OS): Mac OS 是苹果电脑的操作系统，它基于 Unix 操作系统，但具有一些不同的特性，如图形界面、文件管理器、应用程序等。
- YAML（YAML Ain’t Markup Language）是一种直观的数据序列化格式，用于配置文件、数据交换、数据存储等场景。它是一个人类可读的标准，易于与人类和机器进行交互。
    - YAML 语言的设计目标是让数据结构更加直观，并且容易阅读和书写。与 XML 或 JSON 相比，YAML 提供了更加简洁的语法，同时保持了数据结构的灵活性。YAML 支持各种数据类型，包括标量、序列（列表）、映射（字典）、对象等。
- thrussh_keys: 一个用于处理 SSH 密钥的 Rust 库。
- thrussh: 一个用于实现 SSH 服务器的 Rust 库。
- tokio: 一个用于构建异步应用程序的 Rust 库。
- futures: 一个用于处理异步任务的 Rust 库。
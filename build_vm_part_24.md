# 24-SSH 服务器：第二部分 - 你想构建一个语言虚拟机吗？

- [24-SSH 服务器：第二部分 - 你想构建一个语言虚拟机吗？](#24-ssh-服务器第二部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [重新开始](#重新开始)
    - [更新 CLI 选项](#更新-cli-选项)
        - [绑定、主机和端口](#绑定主机和端口)
            - [TCP 和 UDP](#tcp-和-udp)
            - [接口](#接口)
            - [端口](#端口)
            - [绑定](#绑定)
    - [回到 CLI 选项](#回到-cli-选项)
        - [解析选项](#解析选项)
    - [启动服务器](#启动服务器)
        - [远程模块](#远程模块)
            - [客户端](#客户端)
            - [REPL](#repl)
                - [通道](#通道)
            - [总结 REPL 的变化](#总结-repl-的变化)
            - [回到客户端](#回到客户端)
    - [演示 (Demonstration)](#演示-demonstration)
    - [安全](#安全)
    - [结束](#结束)
        - [专有名词及注释](#专有名词及注释)

## 引言

计划有变。我花了好几个小时与 `thrussh` 搏斗 (`fighting`)，试图让 SSH 工作。密钥交换 (`exchange`) 失败了，我不知道为什么。结果发现，即使他们的示例客户端/服务器在我尝试时也无法工作。尽管 (`Despite`) 我花了很多时间研究源代码，但我找不到问题的原因。这个 crate 非常依赖 `futures`，这使得程序流程对我来说很难跟踪。我相信世界上某个地方肯定有人能够毫无问题地跟踪基于 `futures` 的异步，但不是我。

鉴于此 (`In light of this`)，我决定回归老派做法 (`I decided to go old school`)。我将保留之前的教程部分；我认为看到项目这方面的内容也很重要。

不得不放弃行不通的东西并转向其他东西。(`Having to scrap something that doesn’t work out and pivot to something else.`)

那我们将做什么？我们将做一个简单的 TCP 服务器，每个连接都生成 (`spawns`) 一个操作系统线程。Iridium 的远程访问功能并不打算处理成百上千个用户，所以从性能角度来看这没问题。唯一的难点 (`sticking point`) 是加密 (`encryption`)。SSH 本可以让我们有加密连接。我们将不得不在套接字服务器之上实现这一点，但我们会在另一个教程中介绍。

## 重新开始

我们将从这里重新开始：<https://gitlab.com/subnetzero/iridium/tags/0.0.22>. 我们还没有添加任何 thrussh 的依赖。

## 更新 CLI 选项

首先 (`First up`)，让我们在 `src/bin/cli.yml` 中添加一些选项。用户应该能够启用远程访问功能，指定要绑定的主机和端口。

### 绑定、主机和端口

如果你知道什么是 TCP 端口，以及绑定到接口和端口的含义，请随意跳过这一节。如果你不知道，请继续阅读！这将给你一个快速的概述 (`overview`)。

#### TCP 和 UDP

大多数计算机之间的网络通信都使用 (`utilize`) TCP (传输控制协议 (transmission control protocol)) 或 UDP (用户数据报协议 (user datagram protocol))。更高层次的抽象建立在这些之上。例如，HTTP 在底层使用 TCP。这是一个复杂的话题，所以现在，请记住这两点：

- TCP 是一个专用连接，保证按顺序交付数据。把它想象成一次电话交谈。

- UDP 是即发即忘的。你向服务器发送一个数据包，它可能会到达，也可能不会；你不会知道它是否到达。把它想象成邮寄一封信。

对于我们的需求来说，TCP 是最好的选择。

#### 接口

如果服务器在网络中，它正在使用一个 _接口_。这是硬件上网络电缆 (`cable`) 插入的部分，以及它在操作系统中的抽象。这些接口有一个 _IP 地址_，唯一地识别网络上的该服务器。它们看起来像 `192.168.1.10`。每个字段的范围可以从 0 到 255。

| | |
|---|---|
| 注意 | 是的，我知道，有 IPv6 等。我们现在将跳过那个。|

如果你使用的是 Mac 或 Linux 计算机，你可以通过输入 `ifconfig -a` 来查看你的接口。一个例子是：

```sh
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    ether 8c:85:90:7a:53:04
    inet6 fe80::462:3eb0:435b:7753%en0 prefixlen 64 secured scopeid 0x8
    inet 192.168.1.34 netmask 0xffffff00 broadcast 192.168.1.255
    nd6 options=201<PERFORMNUD,DAD>
    media: autoselect
    status: active
```

接口名称是 `en0`，它的 _IP 地址_ 是 `192.168.1.34`。

#### 端口

Linux 服务器有一系列 _端口_，程序可以 _绑定_ 到这些端口。这些范围从 1 到 65535。把它们想象成电话分机。1 到 1024 之间的端口由操作系统保留使用，只有 `root` 用户可以绑定到这些端口。上面的端口可以由你的程序使用。

#### 绑定

程序可以 _绑定_ 到一个 _接口_ 和 _端口_ 的组合。例如，HTTP 服务器绑定到端口 80。HTTPS 绑定到端口 443。一旦程序这样做了，网络上的外部程序就可以访问它。当你通过网络向不同的机器发送请求时，目标 IP 和端口号被包括在内。然后操作系统就知道如何将该消息路由到监听该接口和端口的程序。

## 回到 CLI 选项

用户需要能够指定 Iridium 应该监听的接口和端口。我们这样添加它们：

```yaml
- ENABLE_REMOTE_ACCESS:
    help: Enables the remote server component of Iridium VM
    required: false
    takes_value: false
    long: enable-remote-access
    short: r
- LISTEN_PORT:
    help: Which port Iridium should listen for remote connections on. Defaults to 2244.
    required: false
    takes_value: true
    long: bind-port
    short: p
- LISTEN_HOST:
    help: Which address Iridium should listen for remote connections on. Defaults to "127.0.0.1".
    required: false
    takes_value: true
    long: bind-host
    short: h
```

第一个选项决定 (`determines`) 是否启用远程访问。默认情况下，它是禁用的。我们希望尽可能地 **默认安全**。

### 解析选项

下一步是检查是否启用了远程访问，如果是，用户希望使用哪个主机和端口。打开 `src/bin/iridium.rs` 并在这里放置这个代码块：

```rust
if matches.is_present("ENABLE_REMOTE_ACCESS") {
    let port = matches.value_of("LISTEN_PORT").unwrap_or("2244");
    let host = matches.value_of("LISTEN_HOST").unwrap_or("127.0.0.1");
    start_remote_server(host.to_string(), port.to_string());
}
```

它必须放在 `let matches = App::from_yaml(yaml).get_matches();` 这行下面。这段代码的作用是：

> 1. Check if the --enable-remote-access flag was passed when starting iridium
> 2. Extract the port if provided, or default to 2244 if one was not
> 3. Extract the host if provided, or default to 127.0.0.1 if one was not
> 4. Call the function that starts the remote listener in a background thread

1. 检查启动 iridium 时是否传递了 `--enable-remote-access` 标志
2. 如果提供了端口，则提取端口，否则默认为 2244
3. 如果提供了主机，则提取主机，否则默认为 127.0.0.1
4. 调用在后台线程中启动远程侦听器的函数

| | |
|---|---|
| 重要 | 127.0.0.1 是一个特殊的 IP 地址。每个实现 IPv4 的计算机都有它。你可能听说过它被称为回环 (`loopback`) 地址或 localhost。实际上，如果你在终端中输入 `ping localhost`，你应该看到它正在 ping 那个 IP 地址。这个地址仅指 _本地计算机_。这意味着如果你绑定到 127.0.0.1，你只能从 _同一台计算机_ 访问它。没有来自计算机外部的数据包可以到达 127.0.0.1。</br></br>还有另一个特殊地址 `0.0.0.0`，这意味着 _监听所有可能的_。如果计算机有多个公共 IP，你可以通过任何一个 IP 访问 Iridium（或任何其他程序）。</br></br>默认绑定到 127.0.0.1 是另一个默认安全的选择。|

## 启动服务器

让我们来看看启动 TCP 服务器的函数定义：

```rust
fn start_remote_server(listen_host: String, listen_port: String) {
    let _t = std::thread::spawn(move || {
        let mut sh = iridium::remote::server::Server::new(listen_host, listen_port);
        sh.listen();
    });
}
```

这里有几点需要注意：

> 1. We start a separate thread to handle connections. This is so that the user can continue to interact with the terminal/CLI version, while other users can remote in.
> 2. We create a new Server (we’ll see that next) that is defined in another module, passing it our host and port
> 3. We call listen on that server.

1. 我们启动一个单独的线程来处理连接。这样用户就可以继续与终端/CLI 版本交互，而其他用户可以远程登录。
2. 我们创建了一个新的 Server（我们接下来会看到），在另一个模块中定义，传递给它我们的主机和端口
3. 我们调用该服务器的 listen

### 远程模块

我们将把我们的逻辑放在一个新模块 `src/remote` 中。在其中，你需要 3 个文件：`mod.rs`、`client.rs` 和 `server.rs`。`mod.rs` 很简单：

```rust
pub mod server;
pub mod client;
```

让我们看看 `server.rs` 中有什么：

```rust
use std::io::BufReader;
use std::net::TcpListener;
use std::thread;

pub struct Server {
    bind_hostname: String,
    bind_port: String,
}
```

这就是我们的服务器！现在它没有太多内容，对吧？现在，我们存储了主机名 (`hostname`) 和端口 (`port`)，仅此而已。我们稍后会添加更多内容 (`stuff`)。现在实现稍微复杂 (`complex`) 一些：

```rust
impl Server {
    pub fn new(bind_hostname: String, bind_port: String) -> Server {
        Server {
            bind_hostname,
            bind_port,
        }
    }

    pub fn listen(&mut self) {
        println!("Initializing TCP server...");
        let listener = TcpListener::bind(self.bind_hostname.clone() + ":" + &self.bind_port).unwrap();
        for stream in listener.incoming() {
            let stream = stream.unwrap();
            thread::spawn(|| {
                let mut client = Client::new(stream);
                client.run();
            });
        }
    }
}
```

`new` 函数很简单 (`straightforward`)，所以我不会再详细讲解。`listen` 是功能所在。让我们逐行讲解：

```rust
let listener = TcpListener::bind(self.bind_hostname.clone() + ":" + &self.bind_port).unwrap();
```

这创建了 _监听套接字_ (_`listening socket`_)。当外部用户尝试连接时，这就是他们与之交谈的对象。注意我们如何创建一个 "hostname:port" 形式的字符串。这是指定你想要监听的主机和端口的通常方式。但是并不是保证 (`guaranteed`) 成功的。例如，如果你尝试绑定到其他程序正在监听的端口，它会失败。所以我们希望去掉 unwrap() 并更优雅地处理这种情况。

```rust
for stream in listener.incoming() {
```

监听套接字有一个函数 `incoming`，它将一直 _阻塞_ 直到有连接尝试。它是一个无限循环。当有人连接时，我们得到一个流对象 (`stream object`)：

```rust
let stream = stream.unwrap();
```

这是我们与连接的客户端的 "连接"。我们可以使用它发送和接收数据。最后：

```rust
thread::spawn(|| {
    let mut client = Client::new(stream);
    client.run();
});
```

每当有新客户端连接时，我们创建一个新的 Client 结构体，并调用 `run()`。我们的监听套接字循环是完整的，它回到等待更多连接的状态。

#### 客户端

让我们看看 `client.rs` 中有什么：

```rust
use std::io::{BufRead, Write, Read};
use std::io::{BufReader, BufWriter};
use std::net::TcpStream;
use std::sync::mpsc::{Sender, Receiver};
use std::sync::{mpsc};
use std::thread;
use repl;

pub struct Client {
    reader: BufReader<TcpStream>,
    writer: BufWriter<TcpStream>,
    raw_stream: TcpStream,
    repl: repl::REPL,
}
```

每个客户端都有三个与我们在服务器中看到的相同的 `stream` 副本，以及他们自己的 REPL。你可能想知道为什么我们复制了流。这是因为我们可以将它们包装在 Rust 的 BufReader 和 BufWriter 中，这使我们能够在比字节更高的抽象级别上发送和接收数据。原始流 (`raw_stream`) 在这里是为了创建另外两个。我保证这一会儿就会讲清楚 (`make sense`)。=)

让我们看看实现：

```rust
impl Client {
    pub fn new(stream: TcpStream) -> Client {
        // TODO: Handle this better
        let reader = stream.try_clone().unwrap();
        let writer = stream.try_clone().unwrap();
        let mut repl = repl::REPL::new();

        Client {
            reader: BufReader::new(reader),
            writer: BufWriter::new(writer),
            raw_stream: stream,
            repl: repl
        }
    }
    // 更多函数...
}
```

因为我们需要一个读取器和一个写入器，我们使用流的克隆能力来创建额外的连接。如果 Rust 有自引用结构体 (`self-referential`) (<https://internals.rust-lang.org/t/improving-self-referential-structs/4808>)，这会更干净，但还没有。原始流保留在那里，以防我们将来出于某种原因需要更多克隆。

| | |
|---|---|
| 重要 | 这些并不是真正的额外网络连接。它更像是指向同一个流的多个指针。</br>(`These are not really additional network connections. It’s more like multiple pointers to the same stream.`)|

在 `client.rs` 中还有几个函数：

```rust
fn w(&mut self, msg: &str) -> bool {
    match self.writer.write_all(msg.as_bytes()) {
        Ok(_) => {
            match self.writer.flush() {
                Ok(_) => {
                    true
                }
                Err(e) => {
                    println!("Error flushing to client: {}", e);
                    false
                }
            }
        }
        Err(e) => {
            println!("Error writing to client: {}", e);
            false
        }
    }
}
```

这是一个通用的写入函数。它接受任何字符串，将其转换为字节，并使用我们的 `BufWriter<TcpStream>` 将其发送给客户端。我们调用 flush 以确保它被发送，而不是在某个时间后悬挂。

接下来，我们有：

```rust
fn write_prompt(&mut self) {
    self.w(repl::PROMPT);
}
```

这只是一个方便的函数，将我们的 `>>>` 提示符写给用户。

#### REPL

Client 结构体中还有两个函数，但我们先跳到 repl 模块看看那里的变化，以便提供上下文。打开 `src/repl/mod.rs`。你会注意到添加了一些静态字符串：

```rust
pub static REMOTE_BANNER: &'static str = "Welcome to Iridium! Let's be productive!";
pub static PROMPT: &'static str = ">>> ";
```

这只是为了方便，所以我们可以在其他模块中使用它们。

在 REPL 结构体中，我们添加了两样东西：

```rust
pub struct REPL {
    command_buffer: Vec<String>,
    vm: VM,
    asm: Assembler,
    scheduler: Scheduler,
    pub tx_pipe: Option<Box<Sender<String>>>,
    pub rx_pipe: Option<Box<Receiver<String>>>
}
```

看到 `tx_pipe` 和 `rx_pipe` 了吗？它们现在在构造函数中创建：

```rust
pub fn new() -> REPL {
    let (tx, rx): (Sender<String>, Receiver<String>) = mpsc::channel();
    REPL {
        vm: VM::new(),
        command_buffer: vec![],
        asm: Assembler::new(),
        scheduler: Scheduler::new(),
        tx_pipe: Some(Box::new(tx)),
        rx_pipe: Some(Box::new(rx))
    }
}
```

##### 通道

这些只是普通的 Rust `mpsc` 通道。与我们的 REPL 直接通过 `println!` 写入 `stdout` 不同，它将写入 `tx_pipe`。任何拥有 `rx_pipe` 端的都可以监听 REPL 的输出。

如果你查看各种 REPL 函数，你会看到我已经将 `println!` 宏替换为类似这样的东西：

```rust
self.send_message(format!("Please enter the path to the file you wish to load: "));
```

REPL 现在有两个辅助函数：

```rust
pub fn send_message(&mut self, msg: String) {
    match &self.tx_pipe {
        Some(pipe) => {
            pipe.send(msg+"\n");
        },
        None => {}
    }
}
pub fn send_prompt(&mut self) {
    match &self.tx_pipe {
        Some(pipe) => {
            pipe.send(PROMPT.to_owned());
        },
        None => {}
    }
}
```

还有一个名为 `run_single` 的新函数：

```rust
pub fn run_single(&mut self, buffer: &str) -> Option<String> {
    if buffer.starts_with(COMMAND_PREFIX) {
        self.execute_command(&buffer);
        return None;
    } else {
        let program = match program(CompleteStr(&buffer)) {
            Ok((_remainder, program)) => {
                Some(program)
            }
            Err(e) => {
                self.send_message(format!("Unable to parse input: {:?}", e));
                self.send_prompt();
                None
            }
        };
        match program {
            Some(p) => {
                let mut bytes = p.to_bytes(&self.asm.symbols);
                self.vm.program.append(&mut bytes);
                self.vm.run_once();
                None
            }
            None => {
                None
            }
        }
    }
}
```

这是因为旧的 `run` 函数使用了一个无限循环。如果远程客户端调用它，他们将永远得不到响应。

#### 总结 REPL 的变化

我知道这是很多变化，很难吸收，所以这里有一份整齐的项目列表：

> 1. REPLs now send output over a Rust mpsc channel
> 2. The receiver for that channel can be a remote client
> 3. Remote client input is sent to the run_single function

1. REPL 现在通过 Rust mpsc 通道发送输出
2. 该通道的接收器可以是远程客户端
3. 远程客户端输入被发送到 run_single 函数

#### 回到客户端

我们将要查看的最后两个函数是：

```rust
fn recv_loop(&mut self) {
    let rx = self.repl.rx_pipe.take();
    // TODO: Make this safer on unwrap
    let mut writer = self.raw_stream.try_clone().unwrap();
    let t = thread::spawn(move || {
        let chan = rx.unwrap();
        loop {
            match chan.recv() {
                Ok(msg) => {
                    writer.write_all(msg.as_bytes());
                    writer.flush();
                },
                Err(e) => {}
            }
        }
    });
}

pub fn run(&mut self) {
    self.recv_loop();
    let mut buf = String::new();
    let banner = repl::REMOTE_BANNER.to_owned() + "\n" + repl::PROMPT;
    self.w(&banner);
    loop {
        match self.reader.read_line(&mut buf) {
            Ok(_) => {
                buf.trim_right();
                self.repl.run_single(&buf);
            }
            Err(e) => {
                println!("Error receiving: {:#?}", e);
            }
        }
    }
}
```

当调用 `run` 时，它会调用 `recv_loop`。这从客户端的 REPL 中取出 `rx_pipe`，生成一个线程，并不断监听它。当找到输入时（REPL 发送了某些内容），我们将其发送给客户端。

`run` 函数的其余部分是打印横幅 (`banner`)，然后进入无限循环，监听来自远程客户端的命令。

## 演示 (Demonstration)

让我们看看远程访问的实际效果：

```sh
$ iridium --enable-remote-access
Initializing TCP server...
>>> Welcome to Iridium! Let's be productive!
>>>
```

在另一个终端窗口：

```sh
$ telnet localhost 2244
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Welcome to Iridium! Let's be productive!
>>> !registers
Listing registers and all contents:
[
    0,
        // snip a lot more zeros
    0,
    0
]
End of Register Listing
>>>
```

这种设计可以处理更多的客户端，但由于使用的线程数量，它远非高效。我们稍后将改进这一点。

## 安全

你会注意到没有密码，没有用户名，什么都没有。通过网络发送到 Iridium VM 的任何内容都可以被任何能看到网络流量的人阅读。我们需要在这之上添加加密、认证和授权。但这个教程已经够长了。=)

## 结束

本教程到此为止！你可以在这里找到本教程结束时代码应该的样子：<https://blog.subnetzero.io/project/iridium-vm/building-language-vm-part-23/>。我知道我略过了很多变化，但我们正到达不实际这样做的地步。如果你有任何问题，请在评论、聊天、电子邮件中等发表，我非常乐意回答它们。

下次见！

原文出处：[SSH 服务器：第二部分 - 你想构建一个语言虚拟机吗？](https://blog.subnetzero.io/post/building-language-vm-part-24/)
作者名：Fletcher

### 专有名词及注释

- 传输控制协议 TCP(Transmission Control Protocol): 是一个专用连接，保证按顺序交付数据。把它想象成一次电话交谈。
- 用户数据报协议 UDP(User Datagram Protocol): 是即发即忘的。你向服务器发送一个数据包，它可能会到达，也可能不会；你不会知道它是否到达。把它想象成邮寄一封信。
- mpsc (multiple producer, single consumer): 多生产者单消费者。在 Rust 编程语言中，mpsc 代表“多个生产者，单个消费者”，它是一种并发编程模式，用于在多个生产者线程和单个消费者线程之间安全地传递消息。mpsc 是 Rust 标准库的一部分，提供了线程安全的消息队列，允许不同的线程将消息发送到同一个队列中，而只有一个线程可以从这个队列中接收消息。
    - mpsc 的核心组件是 Sender 和 Receiver。Sender 被用于发送消息，而 Receiver 被用于接收消息。在 Rust 中，mpsc 是通过 std::sync::mpsc 模块提供的，它提供了 channel() 函数来创建一个新的消息通道，该函数返回一对 Sender 和 Receiver。
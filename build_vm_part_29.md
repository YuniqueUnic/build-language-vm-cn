# 29-集群：第三部分 - 你想构建一个语言虚拟机吗？

- [29-集群：第三部分 - 你想构建一个语言虚拟机吗？](#29-集群第三部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
        - [mpsc 通道](#mpsc-通道)
            - [简单使用示例](#简单使用示例)
        - [多的部分](#多的部分)
    - [我们的问题](#我们的问题)
        - [管理器](#管理器)
        - [解决 recv() 问题](#解决-recv-问题)
    - [编译成功？](#编译成功)
        - [接受新的 ClusterClient](#接受新的-clusterclient)
        - [将 ClusterClient 添加到管理器](#将-clusterclient-添加到管理器)
        - [你是谁？](#你是谁)
        - [连接测试](#连接测试)
    - [总结](#总结)
        - [专有名词及注释](#专有名词及注释)

## 引言

如果你尝试编译第 28 个教程的代码，你可能会遇到很多关于无法在 mpsc 通道上发送字符串的错误。这背后的原因值得我们详细讨论。(`The reasons behind this is worth a paragraph or three on why.`)

### mpsc 通道

在 Rust 中，我们有一种称为 mpsc 通道的东西，即多生产者单消费者 (`multi-producer single-consumer`)。想象一下，100 个人在和 1 个人交谈。那 1 个人接收并处理每条消息。

这里有一个创建通道的简单示例：

```rust
use std::sync::mpsc;
fn main() {
    let (tx, rx) = mpsc::channel();
}
```

就是这样！不过，有几点需要注意：

1. `tx` 变量是生产者（或传输）端
    1. `The tx variable is the producer (or transmit) side`

2. `rx` 变量是消费者（或接收器）端
    1. `The rx variable is the consumer (or receiver) side`

#### 简单使用示例

```rust
fn main() {
    let (tx, rx) = mpsc::channel();
    tx.send(1).unwrap();
    println!("rx got: {}", rx.recv().unwrap());
}
```

### 多的部分

tx 端和 rx 端有一个很大的区别：你可以做 `let other_sender = tx.clone()`。然后你通过它发送的任何东西都会到达同一个接收器。

处理这种情况的常见方式是让接收器永远循环，类似于：

```rust
thread::spawn(move || {
    loop {
        if let Ok(msg) = rx.recv() {
            print("{}", msg);
        }
    }
});
```

| | |
|---|---|
| 重要 | 我们不能像克隆 tx 端那样克隆 rx 端。我们只能移动它。|

`recv()` 调用会阻塞，所以我们不会浪费 CPU。每当有消息到达时，我们才处理它。

## 我们的问题

我们需要每个虚拟机的中央客户端存储库 (`repository`)。但我们在单独的线程上监听这些客户端。编译器向我们显示的错误表明我们的架构存在根本性的缺陷 (`fundamental flaw`)。

| | |
|---|---|
| 注 | Rust 的所有权模型通常是一个很好的模型，编译器的严格性 (`strictness`) 有助于执行倾向于是合理的架构。|

### 管理器

我们有这个连接管理器结构。目前，一个虚拟机绑定到一个套接字。这意味着每个虚拟机都需要一个管理器。那么，让我们首先解决这个问题。在 `vm.rs` 中，添加一个字段：

```rust
pub connection_manager: Arc<RwLock<Manager>>,
```

注意我们用 Arc 和 RwLock 包装它，以便我们所有的客户端线程都可以访问它。在 `VM` 的 `new()` 函数中，添加这个：

```rust
connection_manager: Arc::new(RwLock::new(Manager::new())),
```

你可能看到一个错误，像这样：

```sh
error[E0277]: `std::sync::mpsc::Sender<std::string::String>` 不能在线程之间安全地共享(cannot be shared between threads safely)
  --> src/scheduler/mod.rs:20:9
   |
20 |         thread::spawn(move || {
   |         ^^^^^^^^^^^^^ `std::sync::mpsc::Sender<std::string::String>` 不能在线程之间安全地共享
error[E0277]: `std::sync::mpsc::Sender<std::string::String>` 不能在线程之间安全地共享
  --> src/scheduler/mod.rs:20:9
   |
20 |         thread::spawn(move || {
   |         ^^^^^^^^^^^^^ `std::sync::mpsc::Sender<std::string::String>` 不能在线程之间安全地共享
   |
```

我们有问题的函数是这个：

```rust
/// 采用一个虚拟机并在后台线程中运行它
pub fn get_thread(&mut self, mut vm: VM) -> thread::JoinHandle<Vec<VMEvent>> {
    thread::spawn(move || {
        let events = vm.run();
        println!("VM Events");
        println!("--------------------------");
        for event in &events {
            println!("{:#?}", event);
        }
        events
    })
}
```

它试图将虚拟机移动到一个线程中，以便它可以开始执行。但虚拟机包含一个管理器，管理器包含 ClusterClients，它们包含 mpsc 通道。记住，我们不能随意将它们跨线程发送。

为了好玩，让我们看看如果我们用 Arc<Mutex<>> 包装通道会发生什么。在 `src/cluster/client.rs` 中，让我们替换：

```rust
rx: Option<Receiver<String>>,
tx: Option<Sender<String>>,
```

with:

```rust
rx: Option<Arc<Mutex<Receiver<String>>>>,
tx: Option<Arc<Mutex<Sender<String>>>>,
```

现在你可能看到这样的错误：

```sh
error[E0599]: 在当前作用域中未找到名为 `recv` 的方法，类型为 `std::sync::Arc<std::sync::Mutex<std::sync::mpsc::Receiver<std::string::String>>>`
  --> src/cluster/client.rs:66:28
   |
66 |                 match chan.recv() {
   |                            ^^^^

error: 由于先前的错误，编译停止

为了获得有关此错误的更多信息，请尝试使用 `rustc --explain E0599`。
error: 无法编译 `iridium`。
warning: 构建失败，等待其他作业完成...
error[E0599]: 在当前作用域中未找到名为 `recv` 的方法，类型为 `std::sync::Arc<std::sync::Mutex<std::sync::mpsc::Receiver<std::string::String>>>`
  --> src/cluster/client.rs:66:28
   |
66 |                 match chan.recv() {
   |                            ^^^^

error: 由于先前的错误，编译停止
```

如果你用 RwLock 替换 Mutex，你会注意到它不起作用。为什么不？嗯，我会从 Rust 文档中引用：(`Well, I will steal from the Rust docs:`)

> This type of lock allows a number of readers or at most one writer at any point in time. The write portion of this lock typically allows modification of the underlying data (exclusive access) and the read portion of this lock typically allows for read-only access (shared access).
> 
> In comparison, a Mutex does not distinguish between readers or writers that acquire the lock, therefore blocking any threads waiting for the lock to become available. An RwLock will allow any number of readers to acquire the lock as long as a writer is not holding the lock.
> 
> The priority policy of the lock is dependent on the underlying operating system's implementation, and this type does not guarantee that any particular policy will be used.
> 
> The type parameter T represents the data that this lock protects. It is required that T satisfies Send to be shared across threads and Sync to allow concurrent access through readers. The RAII guards returned from the locking methods implement Deref (and DerefMut for the write methods) to allow access to the content of the lock.

```text
这种类型的锁允许一定数量的读取器或最多一个写入器在任何时候。这种锁的写入部分通常允许修改底层数据（独占访问），而读取部分通常允许只读访问（共享访问）。

与 Mutex 相比，RwLock 不区分获取锁的读取器或写入器，因此会阻塞任何等待锁变为可用的线程。只要写入器没有持有锁，RwLock 就会允许任意数量的读取器获取锁。

锁的优先级策略取决于底层操作系统的实现，这种类型不保证使用任何特定的策略。

类型参数 T 表示此锁保护的数据。它需要 T 满足 Send 以跨线程共享，并且满足 Sync 以允许通过读取器并发访问。从锁定方法返回的 RAII 守卫实现了 Deref（以及写入方法的 DerefMut），以允许访问锁的内容。
```

最后一段是关键：T（在这种情况下，我们的通道）只有 `Send` 属性。使用 Mutex 赋予它们 `Sync`，让它们可以跨线程移动。

### 解决 recv() 问题

现在让我们看看我们是否可以解决这个问题。在 `src/cluster/client.rs` 中，我们可以将 `recv_loop()` 函数替换为：

```rust
fn recv_loop(&mut self) {
    let chan = self.rx.take().unwrap();
    let mut writer = self.raw_stream.try_clone().unwrap();
    let _t = thread::spawn(move || {
        loop {
            if let Ok(locked_rx) = chan.lock() {
                match locked_rx.recv() {
                    Ok(msg) => {
                        match writer.write_all(msg.as_bytes()) {
                            Ok(_) => {}
                            Err(_e) => {}
                        };
                        match writer.flush() {
                            Ok(_) => {}
                            Err(_e) => {}
                        };
                    }
                    Err(_e) => {}
                }
            }
        }
    });
}
```

由于我们现在有一个 Mutex 围绕 channels，我们需要锁定它以便在其上调用 recv。由于这个客户端是唯一使用这个特定的 `rx` 端的，这没问题。

## 编译成功？

终于！它编译成功了！尽管我们还没有完全达到我们想要的目标。

### 接受新的 ClusterClient

目前，我们在 `src/cluster/server.rs` 中的 `listen` 函数中监听连接：

```rust
pub fn listen(addr: SocketAddr) {
    info!("Initializing Cluster server...");
    let listener = TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        info!("New Node connected!");
        let stream = stream.unwrap();
        thread::spawn(move || {
            let mut client = ClusterClient::new(stream);
            client.run();
        });
    }
}
```

看看我们在那里没有虚拟机。或者管理器？所以我们不能在那里添加它。让我们尝试传入管理器。在 `src/vm.rs` 中的 `bind_cluster_server` 中，我们有这个：

```rust
pub fn bind_cluster_server(&mut self) {
    if let Some(ref addr) = self.server_addr {
        if let Some(ref port) = self.server_port {
            let socket_addr: SocketAddr = (addr.to_string() + ":" + port).parse().unwrap();
            thread::spawn(move || {
                cluster::server::listen(socket_addr);
            });
        } else {
            error!("Unable to bind to cluster server address: {}", addr);
        }
    } else {
        error!("Unable to bind to cluster server port: {:?}", self.server_port);
    }
}
```

让我们改变它：

```rust
pub fn bind_cluster_server(&mut self) {
    if let Some(ref addr) = self.server_addr {
        if let Some(ref port) = self.server_port {
            let socket_addr: SocketAddr = (addr.to_string() + ":" + port).parse().unwrap();
            // 注意我们在这里需要克隆它，然后将其移动到线程中
            let clone = self.connection_manager.clone();
            thread::spawn(move || {
                // 否则，我们会试图从虚拟机中移动整个东西，而不是一个 Arc
                cluster::server::listen(socket_addr, clone);
            });
        } else {
            error!("Unable to bind to cluster server address: {}", addr);
        }
    } else {
        error!("Unable to bind to cluster server port: {:?}", self.server_port);
    }
}
```

当然，我们还需要改变 `src/cluster/server.rs` 中的 `listen` 函数的签名：

```rust
pub fn listen(addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
```

别忘了添加：

```rust
use cluster::manager::Manager;
```

如果你运行 `cargo test`，它现在应该可以编译了。耶！

### 将 ClusterClient 添加到管理器

现在如果我们前往 `src/cluster/server.rs`，我们现在有一个 `Arc<Mutex<Manager>>` 我们可以用来添加客户端。

除了……因为它们刚刚连接，我们还不知道它们的别名。

### 你是谁？

因为我们的 HashMap 使用节点别名作为键，我们需要在那里放一些东西。有两个明显的选择：

1. 我们让 ClusterClient 先发送它的别名
    1. `We have the ClusterClient send its alias first`

2. 我们为别名生成一个随机 UUID，直到客户端向我们发送正确的一个
    1. `We generate a random UUID for the alias, until the client sends us its proper one`

在 `server.rs` 中，注意这一行：

```rust
client.run();
```

我们本可以这样做：

```rust
thread::spawn(move || {
    let mut buf = String::new();
    let mut client = ClusterClient::new(stream);
    // 一旦这个调用成功，我们希望在字符串缓冲区中有节点别名
    let bytes_read = client.read(&buf);
    client.run();
});
```

当然，这意味着我们需要编写节点别名……所以前往 `src/repl/mod.rs`。`ClusterClient` 的 `join_cluster` 函数看起来像这样：

```rust
fn join_cluster(&mut self, args: &[&str]) {
    self.send_message(format!("尝试加入集群..."));
    let ip = args[0];
    let port = args[1];
    let addr = ip.to_owned() + ":" + port;
    if let Ok(stream) = TcpStream::connect(addr) {
        self.send_message(format!("已连接到集群！"));
        let mut cc = cluster::client::ClusterClient::new(stream);
        cc.send_hello();
        if let Some(ref a) = self.vm.alias {
            if let Ok(mut lock) = self.vm.connection_manager.write() {
                lock.add_client(a.to_string(), cc);
            }
        }
    } else {
        self.send_message(format!("无法连接到集群！"));
    }
}
```

让我们尝试添加一个调用 `send_hello`（在 ClusterClient 中定义的），像这样：

```rust
fn join_cluster(&mut self, args: &[&str]) {
    self.send_message(format!("尝试加入集群..."));
    let ip = args[0];
    let port = args[1];
    let addr = ip.to_owned() + ":" + port;
    if let Ok(stream) = TcpStream::connect(addr) {
        self.send_message(format!("已连接到集群！"));
        let mut cc = cluster::client::ClusterClient::new(stream);
        cc.send_hello();
        if let Some(ref a) = self.vm.alias {
            if let Ok(mut lock) = self.vm.connection_manager.write() {
                lock.add_client(a.to_string(), cc);
            }
        }
    } else {
        self.send_message(format!("无法连接到集群！"));
    }
}
```

当然，编译器希望我们改变 `src/cluster/server` 中的 `listen` 函数：

```rust
pub fn listen(addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
    info!("Initializing Cluster server...");
    let listener = TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        info!("New Node connected!");
        let stream = stream.unwrap();
        thread::spawn(move || {
            let mut buf = [0; 1024];
            let mut client = ClusterClient::new(stream);
            let bytes_read = client.reader.read(&mut buf);
            let alias = String::from_utf8_lossy(&buf);
            println!("Alias is: {:?}", alias);
            client.run();
        });
    }
}
```

改变是预先分配 (`pre-allocated`) 的 1024 字节切片，数据被读取到其中。有了所有这些，测试应该可以通过！

### 连接测试

启动两个虚拟机，在一个上执行 `!start_cluster`，在另一个上执行 `!join_cluster`，你可能会看到类似的东西：

```sh
Started cluster server!
>>> Alias is: "\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u
```

## 总结

我们将在下一篇教程中讨论原因。=）我们快完成了！

原文出处：[Clustering: Part 3 - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-29/)
作者名：Fletcher

### 专有名词及注释

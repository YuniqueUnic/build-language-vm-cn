# 31-使集群化变得有意义 - 你想构建一个语言虚拟机吗？

- [31-使集群化变得有意义 - 你想构建一个语言虚拟机吗？](#31-使集群化变得有意义---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [问题](#问题)
    - [第二个问题](#第二个问题)
    - [列出成员](#列出成员)
        - [接收 Hello](#接收-hello)
        - [集群管理器](#集群管理器)
        - [修复](#修复)
    - [结束](#结束)
    - [专有名词及注释](#专有名词及注释)

## 引言

抱歉，大家！我最近很忙，因此有所耽搁。(`Sorry everyone! I’ve been busy, hence the delay`)

在第 29 期教程中，我们让两个节点彼此通信，但我们在屏幕上得到了很多随机文本。让我们找出原因！

## 问题

```sh
>>> !join_cluster 127.0.0.1 2254
Attempting to join cluster...
Connected to cluster!
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:345:21
stack backtrace:
   0: std::sys::unix::backtrace::tracing::imp::unwind_backtrace
   1: std::sys_common::backtrace::print
   2: std::panicking::default_hook::{{closure}}
   3: std::panicking::default_hook
   4: std::panicking::rust_panic_with_hook
   5: std::panicking::continue_panic_fmt
   6: rust_begin_unwind
   7: core::panicking::panic_fmt
   8: core::panicking::panic
   9: <core::option::Option<T>>::unwrap
  10: iridium::cluster::client::ClusterClient::send_hello
  11: iridium::repl::REPL::join_cluster
  12: iridium::repl::REPL::execute_command
  13: iridium::repl::REPL::run
  14: iridium::main
  15: std::rt::lang_start::{{closure}}
  16: std::panicking::try::do_call
  17: __rust_maybe_catch_panic
  18: std::rt::lang_start_internal
  19: std::rt::lang_start
  20: main
```

在 `send_hello` 中发生了什么？我们似乎没有指定别名！我们可以启动一个没有别名的 Iridium 节点。我们的 `ClusterClient` 初始化如下：

```rust
ClusterClient {
    reader: BufReader::new(reader),
    writer: BufWriter::new(writer),
    raw_stream: stream,
    _tx: Some(Arc::new(Mutex::new(tx))),
    rx: Some(Arc::new(Mutex::new(rx))),
    alias: None,
}

```

让 ID 作为 VM 的 UUID 吧。我们可以为 `ClusterClient` 的 `new` 函数添加一个参数，或者我们可以使用更符合构建者模式 (`builder-pattern`) 的方法。我选择构建者模式，因为我喜欢它。

在 `ClusterClient` 中添加一个新函数：

```rust
/// 设置 ClusterClient 的别名并返回它
pub fn with_alias(mut self, alias: String) -> Self {
    self.alias = Some(alias);
    self
}
```

在 `src/vm.rs` 中，我们需要使 id 属性公开：

```rust
pub id: Uuid,
```

现在，在 `src/repl/mod.rs` 中的 `join_cluster` 函数中，我们有这一行：

```rust
let mut cc = cluster::client::ClusterClient::new(stream);
```

让我们将其更改为结合我们的新函数：

```rust
let mut cc = cluster::client::ClusterClient::new(stream).with_alias(self.vm.id.to_string());
```

现在让我们看看会发生什么！

```sh
>>> Alias is: "aa4356b8-f0a0-4250-b5e5-eaabd265e9b7\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}....that goes on for awhile"
```

但我们得到了别名部分！

## 第二个问题

为什么会这样？还记得我们在 `src/cluster/server.rs` 中声明的那个缓冲区吗？让我们来看看这一段：

```rust
thread::spawn(move || {
    let mut buf = [0; 1024];
    let mut client = ClusterClient::new(stream);
    let _bytes_read = client.reader.read(&mut buf);
    let alias = String::from_utf8_lossy(&buf);
    println!("Alias is: {:?}", alias);
    client.run();
});

```

别名只占用了这么多字节，但我们正在将全部 1024 字节转换成一个字符串！尴尬！我们想要的是取一个切片，只使用我们读取的字节数。

可以看到我们甚至有一个方便的变量 `_bytes_read`，它有这个信息。对我们的代码进行轻微修改……

```rust
thread::spawn(move || {
    let mut buf = [0; 1024];
    let mut client = ClusterClient::new(stream);
    let bytes_read = client.reader.read(&mut buf).unwrap();
    let alias = String::from_utf8_lossy(&buf[0..bytes_read]);
    println!("Alias is: {:?}", alias);
    client.run();
});

```

现在让我们再试一次。在服务器端：

```sh
Fletchers-MBP:iridium fletcher$ iridium
>>> Welcome to Iridium! Let's be productive!
!start_cluster
Started cluster server!
```

然后在客户端：

```sh
Welcome to Iridium! Let's be productive!
>>> !join_cluster 127.0.0.1 2254
Attempting to join cluster...
Connected to cluster!
>>>
```

然后回到服务器端：

```sh
>>> Alias is: "57be0d98-7634-411b-944f-0d1934ead78d"
```

耶！我们只读取了收到的字节数。

| | |
|---|---|
| 重要 | 是的，这意味着每次我们接收数据时，我们只覆盖缓冲区中直到某个点的数据。如果我们读取了 512 字节，那么最后的 512 字节将是上一次输入的 512 字节。</br></br> `Yes, this means that each time we receive data, we only overwrite data up to a certain point in the buffer. If we read 512 bytes, then the last 512 bytes would be the 512 bytes from the previous input.`|

## 列出成员

如果我们在用于加入集群的 VM 上运行 `!cluster_members`，我们得到：

```sh
>>> !cluster_members
Listing Known Nodes:
>>> [
    ""
]
```

如果我们在用于启动集群的 VM 上运行它，我们得到：

```sh
!cluster_members
Listing Known Nodes:
>>> []
```

我们需要完成介绍 (`introduction`) 过程！

### 接收 Hello

让我们从接收别名的点开始。在 `src/cluster/server.rs` 中的 `listen` 函数签名中，我们有：

```rust
pub fn listen(addr: SocketAddr, _connection_manager: Arc<RwLock<Manager>>) {
```

将其更改为 `connection_manager`，然后使函数体如下所示：

```rust
pub fn listen(addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
    info!("正在初始化集群服务器...");
    let listener = TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        let mut cmgr = connection_manager.clone();
        info!("新节点已连接！");
        let stream = stream.unwrap();
        thread::spawn(move || {
            let mut buf = [0; 1024];
            let mut client = ClusterClient::new(stream);
            let bytes_read = client.reader.read(&mut buf).unwrap();
            let alias = String::from_utf8_lossy(&buf[0..bytes_read]);
            cmgr.write().unwrap().add_client(alias.to_string(), client);
            let mut client = cmgr.write().get_client(alias.to_string());
            client.run();
        });
    }
}

```

两个重要的事情需要注意：

> 1. We clone the connection_manager whenever we get a new connection. Since this is an Arc, we’re incrementing the reference count. When it goes out of scope, the reference count will go down. We do this because we don’t want to pass in connection_manager to the thread, because we couldn’t make more clones.
>
> 2. Note how we are not calling client.run(); that’s because we pass ownership of the client to the connection manager. We’ll move that logic there.

1. 我们每次有新连接时都会克隆 connection_manager。由于这是一个 Arc，我们正在增加引用计数。当它离开作用域时，引用计数将减少。我们这样做是因为我们不想将 connection_manager 传递给线程，因为我们不能克隆太多。

2. 注意我们没有调用 `client.run()`；那是因为我们将客户端的所有权传递给了连接管理器。我们将把那个逻辑移过去。

### 集群管理器

前往 `src/cluster/manager.rs`。我们的 `add_client` 函数如下所示：

```rust
pub fn add_client(&mut self, alias: NodeAlias, client: ClusterClient) -> bool {
    if self.clients.contains_key(&alias) {
        error!("尝试添加一个已经存在的客户端");
        return false;
    }
    debug!("正在添加 {}", alias);
    client.run();
    self.clients.insert(alias, client);
    true
}

```

现在我们在添加后调用 run。让我们看看这是否有所改变：

```sh
>>> !join_cluster 127.0.0.1 2254
Attempting to join cluster...
Connected to cluster!
```

然后它会挂起。为什么？好吧，记得我们是如何将调用 `run` 移到连接管理器的吗？它没有在单独的线程中启动，所以它阻塞了主线程。

### 修复

我尝试了一些不同的方法，比如在 `add_client` 函数中启动 `run`。但是，由于我们无论如何都在跨线程做事，我们需要将客户端包装在 Arc 中。

前往 `src/cluster/manager`.rs` 并将 `Manager` 结构体更改为：

```rust
#[derive(Default)]
pub struct Manager {
    clients: HashMap<NodeAlias, Arc<RwLock<ClusterClient>>>,
}

```

现在让我们向 `Manager` 添加一个函数：

```rust
pub fn get_client(&mut self, alias: NodeAlias) -> Option<Arc<RwLock<ClusterClient>>> {
    Some(self.clients.get_mut(&alias).unwrap().clone())
}

```

注意：我们稍后需要改进这一点……
(`NOTE: We’ll have to pretty this up later…​`)

接下来我们需要修改 `add_client` 函数：

```rust
pub fn add_client(&mut self, alias: NodeAlias, client: ClusterClient) -> bool {
    if self.clients.contains_key(&alias) {
        error!("尝试添加一个已经存在的客户端");
        return false;
    }
    let client = Arc::new(RwLock::new(client));
    self.clients.insert(alias.clone(), client);
    let cloned_client = self.get_client(alias).unwrap();
    thread::spawn(move || {
        cloned_client.write().unwrap().run();
    });
    true
}

```

我们在这里做的是将客户端包装在 `Arc` 和 `RwLock` 中，将一个引用添加到我们的 HashMap 中，然后在后台线程中启动它的 `run` 函数。最后，我们需要更改 `src/cluster/server.rs` 中的监听函数：

```rust
pub fn listen(addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
    info!("正在初始化集群服务器...");
    let listener = TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        let mut cmgr = connection_manager.clone();
        info!("新节点已连接！");
        let stream = stream.unwrap();
        thread::spawn(move || {
            let mut buf = [0; 1024];
            let mut client = ClusterClient::new(stream);
            let bytes_read = client.reader.read(&mut buf).unwrap();
            let alias = String::from_utf8_lossy(&buf[0..bytes_read]);
            let mut cmgr_lock = cmgr.write().unwrap();
            cmgr_lock.add_client(alias.to_string(), client);
        });
    }
}

```

注意我们这里不再调用 `run`。让我们看看现在如果我们尝试集群会发生什么：

```sh
Fletchers-MBP:~ fletcher$ iridium
Welcome to Iridium! Let's be productive!
>>> !join_cluster 127.0.0.1 2254
Attempting to join cluster...
Connected to cluster!
>>>
```

耶！现在让我们列出成员……

```sh
>>> !cluster_members
Listing Known Nodes:
>>> [
    ""
]
```

如果我们在用于启动集群的 VM 上列出集群成员：

```sh
>>> !cluster_members
Listing Known Nodes:
>>> [
    "65fda5a6-8b30-4cf1-926f-6fec0d842461"
]
```

呼！

## 结束

我们没有在加入者上看到任何节点的原因是，我们还没有编写任何代码来在客户端加入时将其他已知节点发送回客户端。我们将在下一个教程中完成它！

- 原文出处：[https://blog.subnetzero.io/post/building-language-vm-part-31/](https://blog.subnetzero.io/post/building-language-vm-part-31/)
- 作者名：Fletcher

## 专有名词及注释

1. Arc（原子引用计数）是一个智能指针类型，它提供了多线程环境下的引用计数功能。Arc 是 std::sync::Arc 模块的一部分，是 Rust 标准库的一部分。
   - Arc 的主要用途是在多个线程之间共享数据，同时确保数据在所有引用它的线程都停止使用后才会被清理。Arc 通过内部引用计数机制来管理共享数据的生命周期，类似于 C++ 中的 std::shared_ptr.
2. RwLock（读写锁）是一个智能指针类型，它提供了多线程环境下的读写锁功能。RwLock 是 std::sync::RwLock 模块的一部分，是 Rust 标准库的一部分。
    - RwLock 的主要用途是在多个线程之间安全地共享和修改数据。它允许多个读操作同时进行，但在写操作进行时，所有的读操作和写操作都会被阻塞，直到写操作完成。这允许多个线程并发地读取数据，但写操作需要独占访问。
3. 构建者模式（Builder Pattern）是一种创建型设计模式，用于创建复杂对象。这种模式通常用于以下场景：

    1. 当创建对象的过程需要步骤化处理时。
    2. 当创建对象的过程需要多个步骤，并且这些步骤可能   需要以不同的顺序执行时。
    3. 当需要创建多个相似的对象，这些对象具有相同的基   本部分，但某些部分可能有所不同时。
    4. 当创建过程非常复杂，以至于将创建逻辑与对象的其   他部分混合在一起会导致代码难以维护时。
    在构建者模式中，通常有四个主要角色：
        1. **产品（Product）**：最终要生成的复杂对  象。
        2. **抽象构建者（Abstract Builder）**：定义创   建产品对象的各个步骤的接口。
        3. **具体构建者（Concrete Builder）**：实现抽   象构建者定义的接口，并定义和保持一个用于构建   产品的内部表示。
        4. **指挥者（Director）**：负责安排已有模块的   构建步骤的顺序，并负责创建产品的构建过程。
    使用构建者模式的优点包括：
    - **分离了对象的创建和表示**：构建者模式可以很好    地控制对象的创建过程，使得创建过程更加清晰。
    - **易于控制创建过程**：通过指挥者，可以控制构建    过程，改变构建步骤的顺序，甚至可以创建出部分构建    的产品。
    - **易于复用**：因为构建者独立于产品类，所以可以    很方便地更换产品类。
    - **易于扩展**：可以很容易地增加新的构建者，从而    创建出不同的产品。
    构建者模式的缺点可能包括：
    - **增加复杂性**：如果产品不复杂，使用构建者模式    可能会增加不必要的复杂性。
    - **难以适应变化**：如果产品的内部表示经常变化，那么构建者类也需要相应地改变，这可能会导致维护成    本增加。
    在实际应用中，构建者模式通常用于创建复杂的对象，如创建一个包含多个组成部分的复杂图形用户界面（GUI）应用程序。通过构建者模式，可以一步一步地构   建 GUI 的各个部分，最后组合成一个完整的界面。

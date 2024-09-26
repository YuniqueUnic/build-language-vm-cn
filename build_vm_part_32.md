# 32-更多集群化？！- 你想构建一个语言虚拟机吗？

- [32-更多集群化？！- 你想构建一个语言虚拟机吗？](#32-更多集群化--你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [全网格网络](#全网格网络)
        - [优点](#优点)
        - [缺点](#缺点)
        - [心跳信号](#心跳信号)
        - [其他模型](#其他模型)
        - [进入代码](#进入代码)
        - [协议](#协议)
            - [空白画布](#空白画布)
            - [但是对于列表……](#但是对于列表)
            - [这听起来很繁琐](#这听起来很繁琐)
        - [任务](#任务)
            - [消息枚举](#消息枚举)
            - [序列化](#序列化)
            - [生成消息](#生成消息)
            - [问候确认](#问候确认)
        - [接收消息](#接收消息)
    - [结束](#结束)
    - [原文出处及作者信息](#原文出处及作者信息)
    - [专有名词及注释](#专有名词及注释)

## 引言

大家好，我们又见面了！在本教程中，我们将继续进行集群化的工作。在上一个教程结束时，我们已经让加入节点发送了问候消息，服务器节点也将其添加到了列表中。接下来的任务包括：

1. 回复一个问候
    1. `Send a hello back`
2. 向新加入者发送所有已知节点的列表
    1. `Send a list of all known nodes to the new joiner`

## 全网格网络

还记得我提到我们将要实现全网格网络吗？我觉得提供一个插图可能会很有帮助，所以下面是更多美观的文本艺术！

```text
         ┌───────────────────────────────┐
         │                               │
         │                               ▼
   ┌───────────┐                   ┌───────────┐
   │           │                   │           │
   │           │                   │           │
┌─▶│   VM 1    │         ┌─────────│   VM 2    │
│  │           │         │         │           │
│  │           │         │         │           │
│  └───────────┘         │         └───────────┘
│        │               │               ▲
│        │               │               │
│        │               ▼               │
│        │         ┌───────────┐         │
│        │         │           │         │
│        │         │           │         │
│        └────────▶│   VM 3    │─────────┘
│                  │           │
│                  │           │
│                  └───────────┘
│                        │
│                        │
└────────────────────────┘
```

看到每个节点是如何连接到每个其他节点的吗？这有优点也有缺点：

### 优点

> 1. Relatively simple from a coding standpoint. You just send the entire list to any new joiner. When one leaves, everyone knows about it and removes it from their list.
> 
> 2. With so many connections, the cluster has a high level of resiliency to transient network issues.
> 
> 3. Full-mesh topologies are easy to think about

1. 从编码角度来看相对简单。你只需将完整列表发送给任何新加入者。当一个节点离开时，每个人都知道并将其从列表中移除。

2. 有如此多的连接，集群对瞬态网络问题有很强的韧性。

3. 全网格拓扑结构易于理解。

### 缺点

> 1. TCP Connections to other nodes take a surprising amount of resources. In terms of memory, this can be a few hundred to a few thousand bytes.
>
> 2. Heartbeats

1. 与其他节点的 TCP 连接需要惊人的资源量。在内存方面，这可能是几百到几千字节。

2. 心跳信号

最后一个需要更详细的解释，因此它有一个小节 (`subsection`)！

### 心跳信号

`Heartbeats`

在上面的图表中，VM2 如何知道 VM1 正在运行？它们定期互相发送消息以通讯它们的状态。VM2 也对 VM3 这样做。VM1 发送给 VM2，然后……​好吧，你懂的。

你添加的节点越多，仅仅为了集群运行所需的心跳信号就越多。历史上，Erlang 集群的限制大约是 50 个节点。然而，一个重要的认识是，对于这 50 个节点的 _心跳信号_ 而言，它们是 2 CPU、2 GB RAM 的小机器，还是 128 CPU 5 TB RAM 的怪物，这并不重要。这是你可以结合垂直和水平扩展并获得很多收益的情况之一。

### 其他模型

是的，还有其他模型可以让我们扩展到大约 50 个节点以上。但我们现在不会使用它们。

然而。(`Yet.`)

这些实现起来更复杂，目前这个模型对我们来说已经足够了。

### 进入代码

我们的大部分工作将在 `src/cluster/client.rs` 文件中进行。我们有 `send_hello` 函数，这很好。然而，接收它的函数是 `recv_loop` 函数，如下所示：

```rust
pub fn run(&mut self) {
    self.recv_loop();
    let mut buf = String::new();
    loop {
        match self.reader.read_line(&mut buf) {
            Ok(_) => {
                buf.trim_right();
            }
            Err(e) => {
                println!("Error receiving: {:#?}", e);
            }
        }
    }
}
```

注意在我们的 `run` 函数中，我们进入了一个无限循环，试图从套接字读取。目前，我们所做的只是修剪缓冲区 (`trim the buffer`)，但我们实际上需要处理消息。这意味着我们需要一个协议！

### 协议

`Protocol`

协议就是两个事物之间交流的明确方式。我们可以使用 JSON，来回发送。我们可以使用 YAML，Protobuf，Capnproto，或者许多其他。由于我们这样做是为了学习低级编码，让我们制定自己的简单协议。

#### 空白画布

`The Blank Canvas`

我们是\_ârtįstés！我们可以做到我们想做的任何事！让我们首先在消息前面分配 (`allocating`) 1 个字节来指示消息类型。这给了我们 2^8，或 256 种消息类型。

它看起来像这样：\[[0, 0, 0, 0, 0, 0, 0, 0]\]。那是 8 位，或 1 字节。如果我们将其视为 `u8`，它等于 0。

发送一个包含所有已知节点列表的消息怎么样？为什么，那可以是数字 1：\[[0, 0, 0, 0, 0, 0, 0, 1]\]。

#### 但是对于列表……

这就是我们必须扩展我们的协议的地方。我们可以说跟随的下一个字节包含集群中的节点数。如果它是一个三节点集群，那么我们将有：

```text
[0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 1]
```

如果我们的接收者看到类型为 1 的消息，那么我们编码逻辑告诉它预计在下一个字节中接收节点数。之后，节点可以发送它的完整列表。

假设我们为每个节点使用了一个 UUID。我们知道一个 UUID 占用 128 位。作为一个新加入集群的节点，当我们接收到类型为 1 的消息，后面跟着 3 时，我们知道我们应该期待 128 * 3 更多的位。

#### 这听起来很繁琐

`Well This Sounds Tedious`

嗯，是的。幸运的是 (`Happily`)，我们可以使用 Rust 来序列化数据，很容易做到这一点。Rust 中用于此的重量级工具被称为 `serde`。我们将使用它将我们的消息序列化为称为 `bincode` 的东西，这是一种紧凑的 Rust 特定编码方案 (`Rust-specific encoding scheme`)。

| | |
|---|---|
| 注意 | `serde` 支持 TOML, JSON, YAML 和许多其他。我们使用 bincode 是为了便于使用和紧凑。|

### 任务

1. 定义一个消息枚举
    1. `Define a message enum`
2. 编写一个消息解释器
    1. `Write a message interpreter`

#### 消息枚举

如果你看一下 `src/cluster/message.rs`，你会看到我们已经开始了这个工作。它看起来像这样：

```rust
pub enum IridiumMessage {
    Hello {
        alias: String,
    },
    HelloAck {
        alias: String,
        nodes: Vec<(String, String, String)>,
    },
}
```

注意我们如何发送包含三个字符串的元组向量。第一个是别名，第二个是 IP，第三个是端口。

| | |
|---|---|
| 注意 | 我们可以在这里利用类型别名来使这个更清晰。如果愿意，可以尝试，并在需要帮助时查看源代码。|

我们需要在 Cargo.toml 中添加三个依赖项：

```toml
bincode = "1.0.1"
serde = "1.0.80"
serde_derive = "1.0.80"
```

- `bincode` 添加了编码功能，
- `serde` 添加了核心序列化功能，
- `serde_derive` 将使我们能够派生 `Serialize` 和 `Deserialize` 特征，这是 `bincode` 所需要的。

添加这些到 Cargo.toml 后，我们需要在 `src/bin/iridium.rs` 和 `src/lib.rs` 中添加它们，如下所示：

```rust
extern crate bincode;
#[macro_use]
extern crate serde_derive;
extern crate serde;
```

#### 序列化

`Serialization`

现在我们可以改变我们的枚举为：

```rust
#[derive(Serialize, Deserialize, Debug)]
pub enum IridiumMessage {
    Hello {
        alias: String,
    },
    HelloAck {
        alias: String,
        nodes: Vec<(String, String, String)>,
    },
}

```

注意到我们现在可以派生所需的特征。

#### 生成消息

回到 `message.rs`，我们现在可以实现生成和序列化消息的函数。要生成 `hello` 消息，我们可以这样做：

```rust
impl IridiumMessage {
    pub fn hello(alias: &str) -> Result<Vec<u8>> {
        let new_message = IridiumMessage::Hello{
            alias: alias.into()
        };
        serialize(&new_message)
    }
}

```

我们将得到的是包含别名的 `Vec<u8>`，我们可以通过套接字发送它。如果你看一下 `client.rs`，你可以看到我们的第一个 `send_hello` 函数。我们可以用：

```rust
pub fn send_hello(&mut self) {
    match self.alias {
        Some(ref alias) => {
            if let Ok(hello) = IridiumMessage::hello(alias) {
                if self.raw_stream.write(&hello).is_ok() {
                    trace!("Hello sent!");
                } else {
                    error!("Error sending hello");
                }
            }
        },
        None => {
            error!("Node has no ID to send hello");
        }
    }
}

```

这使用我们的 IridiumMessage 枚举来生成和发送消息。

#### 问候确认

我们需要一个函数来响应 `hello`。尝试自己实现它，但如果卡住了，可以查看 `messages.rs`。注意我在 `client.rs` 中添加了以下便利函数来帮助它：

```rust
pub fn alias_as_string(&self) -> Option<String>;
pub fn ip_as_string(&self) -> Option<String>;
pub fn port_as_string(&self) -> Option<String>;
```

### 接收消息

这部分的难点已经完成。我们有一个接收循环就位，它监听传入的数据包。我们所需要的只是一个反序列化器，我们可以在 IridiumMessage impl 中也放入它。

```rust
pub fn process_message(message: &[u8]) -> Result<IridiumMessage> {
    deserialize(message)
}
```

是的，就这么简单。=）

在 `message.rs` 中，我们需要稍微改变我们的 `run` 函数：

```rust
pub fn run(&mut self) {
    self.recv_loop();
    loop {
        let result: bincode::Result<IridiumMessage> = bincode::deserialize_from(&mut self.reader);
        match result {
            Ok(ref message) => {
                match message {
                    &IridiumMessage::HelloAck{ref nodes, ref alias} => {
                        debug!("Received list of nodes: {:?} from {:?}", nodes, alias);
                    },
                    _ => {
                        error!("Unknown message received");
                    }
                }
                debug!("Received message: {:?}", message);
            },
            Err(e) => {
                error!("Error deserializing Iridium message: {:?}", e);
            }
        }
    }
}
```

不要忘记在顶部添加 `use bincode;`！注意我们在这里解码的消息类型上进行匹配。

我们还需要稍微改变 `server.rs` 中的 `listen` 函数：

```rust
pub fn listen(my_alias: String, addr: SocketAddr, connection_manager: Arc<RwLock<Manager>>) {
    info!("Initializing Cluster server...");
    let listener = TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        let tmp_alias = my_alias.clone();
        let mut cmgr = connection_manager.clone();
        info!("New Node connected!");
        let stream = stream.unwrap();
        thread::spawn(move || {
            let mut client = ClusterClient::new(stream);
            let result: bincode::Result<IridiumMessage> = bincode::deserialize_from(&mut client.reader);
            match result {
                Ok(message) => {
                    match message {
                        IridiumMessage::Hello{alias} => {
                            debug!("Found a hello message with alias: {:?}", alias);
                            let mut cmgr_lock = cmgr.write().unwrap();
                            let mut members: Vec<(String, String, String)> = Vec::new();

                            // 现在我们需要以包含其别名的元组向量的列表形式，发送回集群成员列表
                            for (key, value) in &cmgr_lock.clients {
                                if let Ok(client) = value.read() {
                                    let tuple = (key.to_string(), client.ip_as_string().unwrap(), client.port_as_string().unwrap());
                                    members.push(tuple);
                                }
                            }
                            let hello_ack = IridiumMessage::HelloAck {
                                nodes: members,
                                alias: (tmp_alias.clone(), addr.ip().to_string(), addr.port().to_string())
                            };

                            client.write_bytes(&bincode::serialize(&hello_ack).unwrap());
                            cmgr_lock.add_client(alias.to_string(), client);
                        },
                        _ => {
                            error!("Non-hello message received from node trying to join");
                        }
                    }
                },
                Err(e) => {
                    error!("Error deserializing Iridium message: {:?}", e);
                }
            }
        });
    }
}

```

| | |
|---|---|
| 注意 | 这可以清理很多。|

我们的问题在于这是引导过程。一个节点必须说 Hello，然后得到一个节点列表。在这里更容易做这个逻辑，然后一旦我们确定一切都井然有序，我们可以使用客户端函数来发送。

此时，我们 _应该_ 能够启动两个 Iridium VM，让其中一个尝试加入，并看到问候，并得到一个节点列表。让我们试一试！

从我启动集群的 VM：

```sh
Welcome to Iridium! Let's be productive!
>>> !start_cluster
Started cluster server!
>>>  INFO 2018-10-28T03:20:32Z: iridium::cluster::server: Initializing Cluster server...
 INFO 2018-10-28T03:20:41Z: iridium::cluster::server: New Node connected!
DEBUG 2018-10-28T03:20:41Z: iridium::cluster::server: Found a hello message with alias: "b1ff4bc1-0d72-4d1e-95f3-f384b33f5c2e"
```

从我加入原始节点的另一个 VM：

```sh
>>> !join_cluster localhost 2254
Attempting to join cluster...
Connected to cluster!
TRACE 2018-10-28T03:20:41Z: iridium::cluster::message: Generating hello message
TRACE 2018-10-28T03:20:41Z: iridium::cluster::client: Hello sent: [0, 0, 0, 0, 36, 0, 0, 0, 0, 0, 0, 0, 98, 49, 102, 102, 52, 98, 99, 49, 45, 48, 100, 55, 50, 45, 52, 100, 49, 101, 45, 57, 53, 102, 51, 45, 102, 51, 56, 52, 98, 51, 51, 102, 53, 99, 50, 101]
>>> DEBUG 2018-10-28T03:20:41Z: iridium::cluster::client: Received list of nodes: [] from ("", "127.0.0.1", "2254")
DEBUG 2018-10-28T03:20:41Z: iridium::cluster::client: Received message: HelloAck { alias: ("", "127.0.0.1", "2254"), nodes: [] }
```

耶！成功了！我们成功地反序列化了通过网络发送的 bincoded 结构。

因为我们只有两个节点，一个节点不会将自己添加到连接列表中，我们必须单独在别名字段中发送它。这就是为什么节点列表为空。

## 结束

我们越来越接近了！集群化很棘手 (`finnicky`)，但我们几乎快完成了。`listen` 函数变得相当糟糕 (`awful`)，所以我们可能会尽快重构它，而不是稍后 (`so we’ll probably refactor that sooner rather than later.`)。

我们还需要处理断开连接的情况。如果你现在关闭其中一个 VM，你会看到一连串的错误消息。但是，那是另一个教程！

## 原文出处及作者信息

- 原文出处：[https://blog.subnetzero.io/post/building-language-vm-part-32/](https://blog.subnetzero.io/post/building-language-vm-part-32/)
- 作者名：Fletcher

## 专有名词及注释

1. 心跳信号（Heartbeats）在计算机网络和分布式系统中是一个重要的概念，它用于确保各个系统组件之间的通信是活跃的，以及系统组件是可用的。心跳信号通常是一个定期发送的控制消息，用于表明发送者仍然在线并且运作正常。
    1. 在分布式系统中，节点可能会因为各种原因（如网络问题、硬件故障等）变得不可用。通过定期发送心跳信号，系统中的其他节点可以检测到故障节点，并采取相应的措施，比如重试连接、重新分配任务或者从集群中移除故障节点。
    2. 心跳信号还可以用于同步，例如，在主从复制模型中，从节点可以发送心跳信号给主节点，以通知主节点它已经准备好接收新的数据更新。
    3. 在不同的系统和协议中，心跳信号的具体实现可能有所不同。例如，在 TCP/IP 协议中，心跳信号可以通过保持活动（Keep-Alive）机制实现，该机制通过在 TCP 连接上定期发送探测包来保持连接的活跃状态。
2. serde: serde 是一个数据序列化框架，它允许你以声明性的方式将 Rust 数据结构序列化为多种格式，如 JSON、BSON、MessagePack、YAML 等。
    1. serde 本身并不提供具体的序列化逻辑，而是定义了一套 trait（特征），这些特征描述了如何将数据结构序列化和反序列化。开发者可以通过实现这些 trait 来为自定义类型提供序列化逻辑。
3. serde_derive: serde_derive 是 serde 的一个辅助库，它利用 Rust 的宏系统自动为你的数据结构生成 serde 所需的序列化和反序列化代码。
    1. 你只需要在数据结构上添加 #[derive(Serialize, Deserialize)] 属性，serde_derive 就会自动为你生成实现 Serialize 和 Deserialize trait 的代码。这大大简化了序列化自定义类型的工作。
4. bincode: bincode 是一个基于 serde 的二进制序列化框架，它允许你将 Rust 数据结构序列化和反序列化为一种紧凑的二进制格式。
    1. bincode 提供了高效、跨语言的序列化能力，这意味着序列化后的数据可以被其他语言（如 C++、Python 等）正确解析。bincode 的序列化格式是自描述的，这意味着序列化数据中包含了类型信息，因此可以在没有额外类型信息的情况下进行反序列化。
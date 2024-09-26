# 33-集群同步 - 你想构建一个语言虚拟机吗？

- [33-集群同步 - 你想构建一个语言虚拟机吗？](#33-集群同步---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [代码](#代码)
        - [加入消息](#加入消息)
        - [加入生成函数](#加入生成函数)
        - [发送出加入](#发送出加入)
        - [一些便利函数](#一些便利函数)
    - [处理服务器上的加入](#处理服务器上的加入)
        - [客户端哈希更改](#客户端哈希更改)
    - [结束](#结束)
    - [原文出处及作者信息](#原文出处及作者信息)
    - [专有名词及注释](#专有名词及注释)


## 引言

我不知道你们怎么样，但我对这些集群已经感到厌倦了。但是，这次是真的快结束了！我保证 (`promise`)！

在我们上一个教程结束时，我们已经能够发送 bincoded 消息。一个集群成员可以加入另一个集群，并接收到其他节点的列表。

我们的下一个任务是让新节点对它接收到的每个节点建立自己的、独立的 TCP 连接。

## 代码

你可能已经猜到了，我们将主要在 `src/cluster/client.rs` 中工作。

不过，有一件重要的事情需要注意。如果我们有一个包含 10 个节点的集群，我们不希望向每个节点发送 Hello，并得到相同的节点列表 10 次。

### 加入消息

为了处理这个问题，我们可以创建另一种消息类型，它不触发 (`trigger`) 带有集群中所有其他节点的响应。我们可以像这样将其添加到 `message.rs` 中：

```rust
#[derive(Serialize, Deserialize, Debug)]
pub enum IridiumMessage {
    Hello {
        alias: String,
    },
    HelloAck {
        alias: (String, String, String),
        nodes: Vec<(String, String, String)>,
    },
    Join {
        alias: (String, String, String)
    }
}

```

实际上，现在是改变我们使用 (String, String, String) 的好时机。在 `cluster/mod.rs` 中添加：

```rust
type NodeAlias = String;
type NodeIP = String;
type NodePort = String;
type NodeInfo = (NodeAlias, NodeIP, NodePort);

```

现在我们可以改变我们的 IridiumMessage 枚举，使其看起来像：

```rust
#[derive(Serialize, Deserialize, Debug)]
pub enum IridiumMessage {
    Hello {
        alias: NodeAlias,
        port: NodePort,
    },
    HelloAck {
        alias: NodeInfo,
        nodes: Vec<NodeInfo>,
    },
    Join {
        info: NodeInfo,
        port: NodePort,
    },
}

```

更整洁了！同时，注意在 `Hello` 和 `Join` 消息中添加了 `port`。我们不得不发送这些信息的原因非常重要，值得更详细地描述！

| | |
|---|---|
| 重要 | 每个虚拟机监听一个特定的端口，默认为 2254。如果我们有两台物理机器，每台运行一个虚拟机，那么节点 A 可以连接到节点 B 的 IP 和端口 2254。当它这样做时，还涉及到另一个端口：源端口 (`the source port`)。节点 A 不能从端口 2254 发送数据包，因为它正在使用中。所以应用程序选择了一个随机的高编号端口（通常超过 30000）。这些有时被称为临时端口 (`ephemeral ports`)。</br></br>这里是重要的部分：_节点 A 不能在该端口上接收连接！_ 它只是用来从节点 B 接收响应的。所以当我们发布一个节点时，我们希望发布其他节点可以连接到它的端口，而不是其中一个临时端口。</br></br>弄清楚所有这些端口发布花费了很长时间才正确，但现在它勉强 (`ok-ish`) 可以工作。|

### 加入生成函数

现在让我们在 IridiumMessage impl 中添加一个函数来生成加入消息。

```rust
/// 生成加入消息
pub fn join(alias: &str, port: &str) -> Result<Vec<u8>> {
    trace!("生成加入消息！");
    let new_message = IridiumMessage::Join {
        alias: alias.into(),
        port: port.into(),
    };
    serialize(&new_message)
}

```

这里没有什么特别的。只是记得，当我们调用这个函数时，我们给它的端口是节点希望从其他节点接收连接的端口。这是通过 CLI 上的 -P 标志指定的。

### 发送出加入

当然，现在我们需要做一些工作来将这个消息发送给我们收到的所有节点。目前，我们在 `client.rs` 的 `run` 函数中处理这个问题。具体来说，是这部分：

```rust
match message {
    &IridiumMessage::HelloAck {
        ref nodes,
        ref alias,
    } => {
        debug!("从 {:?} 收到节点列表 {:?}", alias, nodes);
    }

```

也就是说，我们只是打印出节点列表。所以让我们写一个函数，向它们中的每一个发送加入消息。目前，我们只是将需要发送加入的代码直接放入函数中。我们很快需要将其分离 (`factor`) 出来，因为这个 run 函数变得相当长。

```rust
let join_message: std::result::Result<std::vec::Vec<u8>, std::boxed::Box<bincode::ErrorKind>>;
if let Some(ref alias) = self.alias_as_string() {
    join_message = IridiumMessage::join(&alias, &self.port_as_string().unwrap());
} else {
    error!("无法获取我自己的别名以发送加入消息给其他集群成员");
    continue;
}
let join_message = join_message.unwrap();
for node in nodes {
    let remote_alias = &node.0;
    let remote_ip = &node.1;
    let remote_port = &node.2;
    let addr = remote_ip.to_owned() + ":" + remote_port;
    if let Ok(stream) = TcpStream::connect(addr) {
        let mut cluster_client = ClusterClient::new(stream, self.connection_manager.clone(), self.bind_port.clone().unwrap());
        cluster_client.write_bytes(&join_message);
        if let Ok(mut cm) = self.connection_manager.write() {
            let client_tuple = (remote_alias.to_string(), cluster_client.ip_as_string().unwrap(), cluster_client.port_as_string().unwrap());
            cm.add_client(client_tuple, cluster_client);
        }
    } else {
        error!("无法与 {:?} 建立连接", node);
    }
}

```

我们不得不对 `ClusterClient` 数据结构进行了一些更改：

```rust
#[derive(Debug)]
pub struct ClusterClient {
    alias: Option<String>,
    pub reader: BufReader<TcpStream>,
    pub writer: BufWriter<TcpStream>,
    pub connection_manager: Arc<RwLock<Manager>>,
    pub bind_port: Option<String>,
    rx: Option<Arc<Mutex<Receiver<String>>>>,
    _tx: Option<Arc<Mutex<Sender<String>>>>,
    pub raw_stream: TcpStream,
}

```

注意添加了 `bind_port` 字段和 `connection_manager` 字段。这是短期内让 `ClusterClients` 访问这些数据的最简单方法。由于 ClusterClient 可以响应消息，它可能需要添加或删除客户端。它还需要知道节点希望接收连接的端口。

### 一些便利函数

如果你查看 `client.rs`，你会注意到我添加了一些便利函数，用于获取本地和远程 IP 和端口作为 `Strings`。

## 处理服务器上的加入

如果你查看 `server.rs`，你会看到大约在第 59 行的这部分：

```rust
// 处理另一个节点发送的加入消息。在这种情况下，我们不想发送回所有已知节点的列表。
IridiumMessage::Join { alias, port } => {
debug!("从别名 {:?} 收到加入消息", alias);
if let Ok(mut connection_manager) = cmgr.write() {
    debug!("将新客户端 {} 添加到连接管理器", alias);
    let client_tuple = (alias.to_string(), client.remote_ip_as_string().unwrap(), port);
    connection_manager.add_client(client_tuple, client);
} else {
    error!("无法将 {} 添加到连接管理器", alias);
}

```

服务器所做的就是添加客户端。

### 客户端哈希更改

在此之前，我们使用别名作为键。由于锁定复杂性，我将键更改为由 `(alias, ip, port)` 组成的元组。这样我们就不需要在获取客户端列表时获取客户端的锁定。这是一个烦人的变通方法，因为每个客户端在后台都在运行一个接收循环并且锁定了自己。我们将需要找到更好的方法来处理这个问题。

## 结束

如果你查看所有的更改，你会看到添加了很多 debug 日志语句和其他一些小东西。不过，本文涵盖的主题是最重要的。如果你查看新的 `scripts/` 目录，你可以找到一个使用 `tmux` 在单台机器上创建测试 3 节点集群的脚本。如果你是在 OSX 上，你可以使用 `homebrew` 安装 `tmux`。我希望能尽快将其转换为 docker compose 设置。

直到下次！

## 原文出处及作者信息

- 原文出处：[https://blog.subnetzero.io/post/building-language-vm-part-33/](https://blog.subnetzero.io/post/building-language-vm-part-33/)
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
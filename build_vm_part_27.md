# 27-集群：第一部分 - 你想构建一个语言虚拟机吗？

- [27-集群：第一部分 - 你想构建一个语言虚拟机吗？](#27-集群第一部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [为什么？](#为什么)
        - [冗余](#冗余)
        - [扩展](#扩展)
        - [其他东西……](#其他东西)
    - [如何做？](#如何做)
    - [集群设计](#集群设计)
        - [术语](#术语)
    - [命名节点](#命名节点)
        - [生成 ID](#生成-id)
        - [特定 ID](#特定-id)
        - [别名](#别名)
        - [服务器到服务器](#服务器到服务器)
        - [虚拟机实例化更改](#虚拟机实例化更改)
        - [REPL 更改](#repl-更改)
    - [耶，更多的线程](#耶更多的线程)
        - [全网格网络](#全网格网络)
            - [环状](#环状)
            - [全网格](#全网格)
            - [传递性](#传递性)
    - [实现](#实现)
        - [监听套接字](#监听套接字)
        - [服务器](#服务器)
        - [客户端](#客户端)
    - [一个新命令](#一个新命令)
        - [专有名词及注释](#专有名词及注释)

## 引言

大家好！教程延迟是因为网站重做（感谢 [Bitdream](https://bitdream.io/)），我希望这看起来不那么糟糕。在此期间，我确实添加了一些功能并修复了一些不会让教程变得有趣的错误，所以你应从最新的 `master` 分支开始本教程。

在本教程中，我们将让虚拟机的不同实例通过 TCP 相互通信。它们不会做太多事情，但至少能够连接。

## 为什么？

好问题！

### 冗余

`Redundancy`

如果你的应用程序在不止一个物理独立的服务器上运行，如果一个失败了，另一个可以继续运行。在 Iridium 虚拟机网络的情况下，让它们相互了解允许智能处理故障。

### 扩展

`Scaling`

如果你遇到流量激增，可以启动运行你的应用程序的更多机器。

### 其他东西……

`Other Stuff…​`

我还有一些其他很酷的想法，但我还不确定它们是否可行。所以我将稍后再讲。=)

## 如何做？

和我们添加多用户支持的方式一样！我们将在节点成员之间打开 TCP 通道。

## 集群设计

`Cluster Design`

这就有点复杂了。分布式计算是一个难题，有很多方法可以实现。当前行业里的热门词汇包括：

- Raft

- Paxos

- 无领导 (Leaderless)

- 一致性哈希 (Consistent hashing)

分布式系统中的许多问题源于构建分布式 _数据库_ (`distributed database`) 系统。由于我们不是在构建那样的东西，我们需要担心的就少了。另外，我们可以直接借鉴 Erlang BEAM 虚拟机的设计！这非常方便。

### 术语

`Terminology`

在我们开始之前，让我们定义一些术语。

1. `节点(Node)` 是一个运行中的 Iridium 虚拟机

2. `主机(Host)` 是运行 Linux 或其他操作系统的物理或虚拟服务器

3. `集群(Cluster)` 是可以在同一共享网络上相互通信的 `节点(Nodes)` 集合

4. `拓扑(Topology)` 是集群如何连接在一起的方式。所有节点都连接到所有其他节点吗？有没有代理节点？

| | |
|---|---|
| 重要 | 一个 `主机` 可以运行多个 `节点`。|

## 命名节点

我们需要能够在集群中唯一识别每个节点。也就是说，我们不希望有两个节点命名为 `Murgatroyd`。这种唯一性很重要，因为我们将能够在节点之间传递 `消息`，其中一部分是能够唯一识别该节点。

### 生成 ID

一个选择是只使用随机 UUID。实际上，我们在虚拟机结构中添加了一个 UUID 字段一段时间了，并且用随机 UUID 填充它。如果我们对此感到满意，那么我们已经完成了这一步！

### 特定 ID

虽然 UUID 很酷，但它们可能，嗯，难以记住。提供 CLI 标志以指定节点 ID 会很好。这样做的问题是有人可能会给 _完全不同的 Iridium 虚拟机_ 相同的名称。这对审计目的不利，是当前系统生成随机 UUID 的一个好处。

### 别名

`Aliases`

所以我们需要的是别名。这是一个用户在启动 Iridium 虚拟机时可以提供的任意字符串。如果他们提供了别名，`节点` 可以通过别名或 UUID 被寻址。

打开 `src/bin/cli.yml`，让我们添加一个可选的 `node-alias` 标志。就在 DATA_ROOT_DIR 选项下面，我们可以添加：

```yaml
- DATA_ROOT_DIR:
    help: Iridium VM 应该存储其数据的根目录。默认为 /var/lib/iridium。
    required: false
    takes_value: true
    long: data-root-dir
- NODE_ALIAS:
    help: 可以用来在网络上引用运行中的虚拟机的别名
    required: false
    takes_value: true
    long: node-alias
```

然后在 `iridium.rs` 中，我们添加一个检查它：

```rust
let alias = matches.value_of("NODE_ALIAS").unwrap_or("");
```

最后，在 `src/vm.rs` 中，我们将新的 `alias` 字段添加到虚拟机结构体：

```rust
pub struct VM {
    /// 模拟硬件寄存器的数组
    pub registers: [i32; 32],
    /// 模拟浮点硬件寄存器的数组
    pub float_registers: [f64; 32],
    /// 跟踪正在执行的字节的程序计数器
    pc: usize,
    /// 正在运行的程序的字节码
    pub program: Vec<u8>,
    /// 用于堆内存
    heap: Vec<u8>,
    /// 用于表示堆栈
    stack: Vec<u8>,
    /// 包含模除操作的余数
    remainder: usize,
    /// 包含最后一次比较操作的结果
    equal_flag: bool,
    /// 循环计数器字段，与 `LOOP` 指令一起使用
    loop_counter: usize,
    /// 包含只读部分数据
    ro_data: Vec<u8>,
    /// 是一个唯一、随机生成的 UUID，用于标识这个虚拟机
    id: Uuid,
    /// 用户可以指定并用于引用节点的别名
    alias: Option<String>,
    /// 保存特定虚拟机的事件列表
    events: Vec<VMEvent>,
    /// 系统报告的逻辑核心数
    pub logical_cores: usize,
}

```

让我们使用构建器类型的函数来传递别名。在虚拟机实现中，我们可以添加这个函数：

```rust
pub fn with_alias(mut self, alias: String) -> Self {
    if alias == "" {
        self.alias = Some(alias)
    } else {
        self.alias = None
    }
    self
}
```

然后在 `src/bin/iridium.rs` 文件中，我们在实例化虚拟机时可以这样做：

```rust
let alias = matches.value_of("NODE_ALIAS").unwrap_or("");
```

这样就解决了别名部分！让我们继续网络部分。让我们向虚拟机结构体添加另一个构建器风格的函数。

```rust
pub fn with_cluster_bind(mut self, server_addr: String, server_port: String) -> Self {
    self.server_addr = Some(server_addr);
    self.server_port = Some(server_port);
    self
}
```

这将使用我们从 CLI 获取的字符串设置字段，如果用户传递了它们。

### 服务器到服务器

这与我们实现多用户 REPL 的方式没有太大不同。我们不想使用相同的套接字 (`socket`)，因为这将是专用于服务器到服务器通信的。所以让我们添加指定 (`dedicated`) 服务器到服务器通信的地址和端口的能力。

让我们在 `cli.yml` 中添加两个更多选项：

```yaml
- SERVER_LISTEN_PORT:
    help: Iridium 应该监听来自其他 Iridium 虚拟机的远程连接的端口。默认为 2254。
    required: false
    takes_value: true
    long: server-bind-port
    short: P
- SERVER_LISTEN_HOST:
    help: Iridium 应该监听来自其他 Iridium 虚拟机的远程连接的地址。默认为 "127.0.0.1"。
    required: false
    takes_value: true
    long: server-bind-host
    short: H
```

然后在 `iridium.rs` 中获取它们的值：

```rust
let server_addr = matches.value_of("SERVER_LISTEN_HOST").unwrap_or("127.0.0.1");
let server_port = matches.value_of("SERVER_LISTEN_PORT").unwrap_or("2254");
```

当然，我们还需要向虚拟机添加更多字段。=)

```rust
// 虚拟机将绑定用于服务器到服务器通信的服务器地址
server_addr: Option<String>,
// 服务器将绑定用于服务器到服务器通信的端口
server_port: Option<String>
```

### 虚拟机实例化更改

`VM Instantiation Change`

为了让虚拟机绑定到套接字，我们需要更改这一点：

变为：

```rust
let mut vm = VM::new().with_alias(alias.to_string()).with_cluster_bind(server_addr.into(), server_port.into());
```

在两个地方。一个是我们在 `INPUT_FILE` 处调用虚拟机（在 `iridium.rs` 中大约第 64 行），另一个是当我们为 `REPL` 创建虚拟机时，大约在第 83 行。

这是一次小重构，以使我们的生活更轻松。

### REPL 更改

第二个也需要我们对 REPL 进行小改动：

```rust
let mut repl = REPL::new(vm);
```

我们现在想传入一个虚拟机实例。在 `repl.rs` 中，这意味着我们需要将签名更改为：

```rust
pub fn new(vm: VM) -> REPL
```

## 耶，更多的线程

是的，我们将为此使用线程。不过，这有一个权衡 (`tradeoff`)。我们还将在一个 _全网格网络_ (_`full-mesh`_) 中使用它。

### 全网格网络

`Full-Mesh Network`

如果我们有一个由 5 个 `节点` 组成的集群，我们可以有

#### 环状

在这种拓扑中，节点连接将如下所示：

```text
A -> B -> C -> D -> E -> A
```

每个 `节点` 连接到另外两个 `节点`。如果我们失去了一个节点，我们仍然有一条连接剩余节点的路径，但这不是最优的。

#### 全网格

全网格拓扑 (`topology`) 将如下所示：

```text
A -> (B, C, D, E)
B -> (A, C, D, E)
C -> (D, A, B, E)
D -> (A, B, C, E)
E -> (A, B, C, D)
```

每个节点都连接到每个其他节点。这具有非常有弹性的 (`resilient`) 优点，但也很贵。随着节点数量的增加，每个节点上所需的连接数量也在增加。它是 O(N!)，其中 N 是 `节点` 的数量。所以不是很好。

不过，目前它会满足我们的需求 (`purposes`)，我们稍后可以通过使用部分网格网络 (`partial-mesh network`) 等技术来优化它。

#### 传递性

`Transitivity`

假设我们有一个由 3 个 `节点` 组成的集群，A、B 和 C。如果 `A → C`，A 和 C 也将连接到 B。或者换句话说，每当一个新节点加入集群时，它会获得集群所有节点的列表，并连接到它们。

## 实现

好的，让我们实现它。首先，我们需要：

> 1. A background thread that listens for connections
> 2. Something that watches for topology changes (nodes entering/leaving the cluster) and responds appropriately

1. 一个后台线程，监听连接

2. 一个监控拓扑变化（节点进入/离开集群）并相应响应的东西

让我们从……开始

### 监听套接字

我们之前做过这个，所以它是一个很好的起点。让我们在名为 `cluster` 的新模块中做这个。在 `src/` 下创建 `cluster/`，然后在那里创建三个文件：`mod.rs`、`client.rs` 和 `server.rs`。让我们从服务器端开始。

### 服务器

这相当简单，和 `remote/server.rs` 代码非常相似：

```rust
use std::net::{TcpListener, SocketAddr};
use std::thread;
use cluster::client::ClusterClient;

pub fn listen(addr: SocketAddr) {
    info!("Initializing Cluster server...");
    let listener =
        TcpListener::bind(addr).unwrap();
    for stream in listener.incoming() {
        info!("New Node connected!");
        let stream = stream.unwrap();
        thread::spawn(|| {
            let mut client = ClusterClient::new(stream);
            client.run();
        });
    }
}

```

我们创建一个监听器并开始监听连接。当有新客户端（在这种情况下是 Iridium 虚拟机）连接时，我们为其生成一个新线程，启动集群客户端，然后回去监听。

### 客户端

这个有点长，所以我将在代码本身中注释解释发生了什么。

```rust
use std::io::{BufRead, Write};
use std::io::{BufReader, BufWriter};
use std::net::TcpStream;
use std::thread;
use std::sync::mpsc::channel;
use std::sync::mpsc::{Receiver, Sender};

/// 这代表加入此服务器的另一个 Iridium 虚拟机。在它们端，我们将是 ClusterClient。
/// 这与 `Client` 的主要区别是我们不需要创建 `REPL`。
pub struct ClusterClient {
    // 用 BufReader 包装流，使其更容易读取
    reader: BufReader<TcpStream>,
    // 用 BufWriter 包装流，使其更容易写入
    writer: BufWriter<TcpStream>,
    // 这些是标准 mpsc 通道。
    // 我们将启动一个线程，监视此通道上来自我们应用程序其他部分的消息
    // 被发送到 ClusterClient
    rx: Option<Receiver<String>>,
    // 如果有东西想要发送东西给这个客户端，它们可以克隆 `tx` 通道。
    tx: Option<Sender<String>>,
    raw_stream: TcpStream,
}

impl ClusterClient {
    /// 创建并返回一个新的 ClusterClient，设置流克隆和 mpsc 通道
    pub fn new(stream: TcpStream) -> ClusterClient {
        // TODO: 处理得更好
        let reader = stream.try_clone().unwrap();
        let writer = stream.try_clone().unwrap();
        let (tx, rx) = channel();
        ClusterClient {
            reader: BufReader::new(reader),
            writer: BufWriter::new(writer),
            raw_stream: stream,
            tx: Some(tx),
            rx: Some(rx)
        }
    }

    // 将消息作为字节写入连接的 ClusterClient
    fn w(&mut self, msg: &str) -> bool {
        match self.writer.write_all(msg.as_bytes()) {
            Ok(_) => match self.writer.flush() {
                Ok(_) => true,
                Err(e) => {
                    println!("Error flushing to client: {}", e);
                    false
                }
            },
            Err(e) => {
                println!("Error writing to client: {}", e);
                false
            }
        }
    }

    // 这是一个后台循环，监视 mpsc 通道上的消息。当它收到一个时，它将其发送到 ClusterClient。
    fn recv_loop(&mut self) {
        // 我们取 rx 通道，以便我们可以将所有权移入线程并循环
        let chan = self.rx.take().unwrap();
        let mut writer = self.raw_stream.try_clone().unwrap();
        let _t = thread::spawn(move || {
            loop {
                match chan.recv() {
                    Ok(msg) => {
                        match writer.write_all(msg.as_bytes()) {
                            Ok(_) => {}
                            Err(_e) => {}
                        };
                        match writer.flush() {
                            Ok(_) => {}
                            Err(_e) => {}
                        }
                    }
                    Err(_e) => {}
                }
            }
        });
    }

    /// 调用以设置 ClusterClient 的主要函数
    pub fn run(&mut self) {
        // 在后台线程中启动 recv_loop
        self.recv_loop();
        let mut buf = String::new();
        // 循环处理传入数据，等待数据。
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
}

```

呼！抱歉代码块这么长。总结一下：

> 1. VM starts up
> 2. A thread binds to a TCP socket and listens for incoming connections
> 3. New connections are handed off to a new thread, that starts their recv loop

1. 虚拟机启动

2. 一个线程绑定到 TCP 套接字并监听传入连接

3. 新连接被交给一个新线程，启动它们的 recv 循环

## 一个新命令

我们还没有启动集群的方法。为此，让我们进入 `src/repl/mod.rs` 并添加一个名为 `!start_cluster` 的新命令。我将把那个实现作为一个有趣的练习留下，但如果卡住了，可以参考源代码：<https://gitlab.com/subnetzero/iridium/tags/0.0.27>

在下一个教程中，我们将处理集群成员之间的消息传递。下次见！

原文出处：[Clustering: Part 1 - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-27/)
作者名：Fletcher

### 专有名词及注释

1. Raft: 是一种用于管理复制日志的一致性算法，旨在提供一种更易于理解的一致性算法。它将一致性算法分解为几个关键问题，并通过简单的状态机和规则来解决这些问题。
2. Paxos: 是一种用于在分布式系统中达成共识的算法。它允许一组节点就某个值达成一致，即使部分节点出现故障。Paxos 算法通常用于分布式系统中的复制状态机。
3. 无领导 (Leaderless):无领导架构是一种去中心化的系统架构，其中没有明确的领导者。在这种架构中，所有节点平等地参与决策和任务分配，避免了单点故障问题。
4. 一致性哈希 (Consistent hashing):是一种分布式哈希算法，用于将数据均匀分布到多个节点上。它通过将哈希值空间视为一个环，并将数据根据哈希值映射到环上的位置，从而实现负载均衡和可扩展性。
5. `节点(Node)` 是一个运行中的 Iridium 虚拟机，在分布式系统中，节点是一个运行中的计算机实例，它可以是一个物理服务器、虚拟机、容器或任何能够执行计算任务和存储数据的设备
6. `主机(Host)` 是运行 Linux 或其他操作系统的物理或虚拟服务器，主机是一个物理或虚拟的服务器，它为节点提供运行环境。在物理服务器的情况下，主机是一个具有处理器、内存、存储和网络接口的计算机。在虚拟化环境中，主机可能是一个运行多个虚拟机的物理服务器。主机通常运行 Linux 或其他操作系统，并提供必要的资源和服务来支持其上的节点。主机的性能和配置直接影响其上运行节点的性能和可靠性。
7. `集群(Cluster)` 是可以在同一共享网络上相互通信的 `节点(Nodes)` 集合，集群是一组节点，它们通过网络连接并协同工作，以提供更高的性能、可靠性和可扩展性。在集群中，节点可以共同执行任务，如数据处理、存储和计算，以实现共同的目标。集群通常用于提高应用程序的可用性和容错能力，确保即使部分节点失败，系统仍能继续运行。
8. `拓扑(Topology)` 是集群如何连接在一起的方式。所有节点都连接到所有其他节点吗？有没有代理节点？拓扑描述了集群中节点之间的连接方式。它定义了数据如何在节点之间流动，以及节点如何相互通信。常见的拓扑结构包括：全连接拓扑，星型拓扑，环形拓扑，树状拓扑

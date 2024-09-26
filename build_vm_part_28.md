# 28-集群：第二部分 - 你想构建一个语言虚拟机吗？

- [28-集群：第二部分 - 你想构建一个语言虚拟机吗？](#28-集群第二部分---你想构建一个语言虚拟机吗)
    - [引言](#引言)
    - [加入集群命令](#加入集群命令)
        - [REPL](#repl)
        - [集群模块](#集群模块)
            - [一点设置](#一点设置)
            - [节点列表](#节点列表)
    - [消息](#消息)
        - [制定协议](#制定协议)
        - [但是……](#但是)
        - [组建一个集群](#组建一个集群)
        - [加入集群](#加入集群)
            - [集群消息](#集群消息)
            - [客户端](#客户端)
            - [测试](#测试)
            - [列出成员](#列出成员)
    - [总结](#总结)
        - [专有名词及注释](#专有名词及注释)

## 引言

大家好，又是我！在我们上一个教程中，我们为 Iridium 虚拟机添加了一个独立的 TCP 服务器。在这个教程中，我们将完成客户端部分，以便两个 Iridium 虚拟机可以相互通信。请确保你从这个标签开始本教程：<https://gitlab.com/subnetzero/iridium/tags/0.0.27。>

| | |
|---|---|
| 重要 | 本文末尾的代码尚未完成。我们还有一个教程要做！|

## 加入集群命令

在本教程中，让我们为我们生活增添一些变化，首先制作一个加入集群的命令。目前，有一个 `!start_cluster` 命令。让我们添加一个 `!join_cluster` 命令。

### REPL

这只是几个步骤：

1. 在 `src/repl/mod.rs` 中，找到名为 `execute_command` 的函数，并为它添加 `!start_cluster`。类似于：`"!join_cluster" ⇒ self.join_cluster(&args[1..]),`

2. 在底部制作一个占位 (`stub`) 函数

占位函数应该看起来像这样：

```rust
fn join_cluster(&mut self, _args: &[&str]) {
    debug!("尝试加入集群...");
}
```

我们当然会边做边添加。

### 集群模块

由于我们正在进行全网格，每个节点都是服务器，并且它作为客户端连接到每个其他节点。这意味着一些事情：

> 1. We need to keep a list of all known nodes
> 2. We need to track the network connectivity to all our connected nodes
> 3. We need to keep the list of nodes in sync across all nodes

1. 我们需要保持所有已知节点的列表

2. 我们需要跟踪我们连接到的所有节点的网络连接

3. 我们需要在所有节点之间保持节点列表同步

| | |
|---|---|
| 注 | 这是我们开始做一些很酷的分布式系统 (`distributed systems`) 工作的地方。=)|

#### 一点设置

1. 如果你还没有，就创建目录 `src/cluster`

2. 创建一个空的 `client.rs` 文件

3. 创建一个空的 `server.rs` 文件

4. 创建一个空的 `mod.rs` 文件

5. 创建一个名为 `manager.rs` 的空文件

在 `mod.rs` 文件中，放置：

```rust
pub mod server;
pub mod client;
pub mod manager;

type NodeAlias = String;
```

#### 节点列表

让我们从这里开始。我们可以制作一个简单的 Manager 结构，由它来处理这些。它将负责 (`responsibility`) 跟踪每个连接的客户端，添加新节点，以及移除失效节点。创建 `src/cluster/manager.rs`，并在其中放置以下内容。由于它是较大的一块 (`chunk`)，我将在代码注释中解释。

```rust
use std::collections::HashMap;

use cluster::client::ClusterClient;
use cluster::NodeAlias;

// 基本结构声明。目前，它有一个简单的 HashMap，按其 NodeAlias 映射 ClusterClients
// 记住：NodeAlias 只是 String 的类型别名。
pub struct Manager {
    clients: HashMap<NodeAlias, ClusterClient>
}

impl Manager {
    /// Manager 的基本构造函数
    pub fn new() -> Manager {
        Manager {
            clients: HashMap::new()
        }
    }

    /// 添加一个客户端，这将是另一个集群成员。我们真的应该返回一些不是布尔值的东西，
    /// 但我们将在我们重新处理错误时重新审视这个问题。
    pub fn add_client(&mut self, alias: NodeAlias, client: ClusterClient) -> bool {
        if self.clients.contains_key(&alias) {
            error!("尝试添加一个已经存在的客户端");
            return false;
        }
        self.clients.insert(alias, client);
        true
    }

    /// 通过别名删除客户端。与上面的布尔值相同。
    pub fn del_client(&mut self, alias: NodeAlias) -> bool {
        self.clients.remove(&alias);
    }
}

// 当然还有一些测试
#[cfg(test)]
mod test {
    use super::Manager;

    fn test_create_manager() {
        let test_manager = Manager::new();
    }
}
```

## 消息

这就是事情开始变得棘手 (`tricky`) 的地方。我们需要一个 _协议_ (_`protocol`_) 来在集群成员之间来回通信。我们只能发送 0 和 1。接收器怎么知道在那个流中的节点别名在哪里，例如？一个协议让我们定义这些东西。

### 制定协议

已经有很多现成的协议。JSON 是其中之一。Protobuf。Flatbuf。我们也可以自己制定一个。

作为一个例子，我们可以说前 64 个字节是节点别名。这有几个影响：

1. 节点别名只能有 64 个字节或更少

2. 如果节点别名少于 64 个字节，我们可能会浪费空间

### 但是……

我开始使用 Cap’n Proto 实现这部分，但最终我认为它对于教程来说太复杂了。所以让我们用另一种方式来做：JSON！但首先，让我们先了解一下组建和加入集群的过程。

### 组建一个集群

集群的启动需要有一个节点来引导整个集群。目前，这意味着运行 `!start_cluster` 命令。将来，我们会通过命令行标志或环境变量来实现这一点。启动集群会将 _特定的 Iridium 虚拟机进程_ 绑定到一个特定 (`particular`) 的端口上，并开始监听数据。

现在，我们已经有了一个集群。虽然它目前只有一个节点，但仍然是一个集群。(`We now have a cluster. A cluster of 1, but still.`)

### 加入集群

现在我们可以启动第二个 Iridium 虚拟机，并使用引导节点的 IP 和端口参数使用 `!join_cluster`。它将连接。现在，这里必须发生几件事情。把它想象成握手 (`handshake`)，或者节点彼此介绍。如果我们的节点被称为 A 和 B，它可能会这样进行：

B："你好 A！我叫 B！"
A："你好 B！我叫 A！这里是我在集群中所知道的所有其他节点！"

当然，如果我们加入加密、认证、授权 (`encryption, authentication, authorization`)，那个交互就会变得更加复杂。在本教程的剩余部分，让我们先让它工作。

#### 集群消息

我们将使用枚举来表示每种不同的消息类型：

```rust
pub enum IridiumMessage {
  Hello{alias: String},
  HelloAck{alias: String, nodes: Vec<(String, String, String)>}
}
```

`Hello` 包含想要加入集群的节点的节点别名。`HelloAck` 用它自己的别名回应，并列出所有节点、它们的 IP 和端口。我们的加入者随后需要遍历这些节点，并自我介绍。但目前，让我们先让加入者制作并发送一个 `Hello`。

继续打开 `src/cluster/message.rs` 并在其中放置：

```rust
pub enum IridiumMessage{
    Hello{alias: String},
    HelloAck{alias: String, nodes: Vec<(String, String, String)>}
}
```

别忘了在 `mod.rs` 中添加 `pub mod message;`。

#### 客户端

打开 `src/cluster/client.rs`。在 ClusterClient impl 中，添加以下函数：

```rust
pub fn send_hello(&self) {
    let msg = IridiumMessage{alias: ....};
}
```

糟糕 (`Crap`)。我们没有在 TcpClient 中存储别名。所以让我们向结构体添加一个字段：

```rust
pub struct ClusterClient {
    alias: String,
    reader: BufReader<TcpStream>,
    writer: BufWriter<TcpStream>,
    rx: Option<Receiver<String>>,
    tx: Option<Sender<String>>,
    raw_stream: TcpStream,
}
```

然后在 `impl` 中，我们需要更改 `new` 函数：

```rust
pub fn new(stream: TcpStream, alias: String) -> ClusterClient {
    // TODO: Handle this better
    let reader = stream.try_clone().unwrap();
    let writer = stream.try_clone().unwrap();
    let (tx, rx) = channel();
    ClusterClient {
        reader: BufReader::new(reader),
        writer: BufWriter::new(writer),
        raw_stream: stream,
        tx: Some(tx),
        rx: Some(rx),
        alias: alias,
    }
}
```

现在让我们前往 (`head over`) `repl/mod.rs`，并向下到 `join_cluster` 方法。我已经在代码中添加了注释。

```rust
fn join_cluster(&mut self, args: &[&str]) {
  self.send_message(format!("尝试加入集群..."));
  // 提取传入的 IP 和端口参数
  let ip = args[0];
  let port = args[1];
  // 将它们转换为我们可以用作 SocketAddr 的形式
  let addr = (ip.to_owned() + ":" + port);
  // 尝试建立实际连接
  if let Ok(stream) = TcpStream::connect(addr) {
      self.send_message(format!("已连接到集群！"));
      // 将远程集群添加到我们的连接集群列表中
      let cc = cluster::client::ClusterClient::new(stream);
      if let Some(ref a) = self.vm.alias {
          self.connection_manager.add_client(a.to_string(), cc);
      }
  } else {
      self.send_message(format!("无法连接到集群！"));
  }
}
```

#### 测试

对于基本测试，我打开一个终端，并垂直分割屏幕。在一个中，我这样做：

```sh
$ iridium --node-alias=node1
欢迎使用 Iridium! 让我们提高生产力!
>>> !start_cluster
正在启动集群服务器!
>>>
```

在另一个中：

```sh
$ iridium --node-alias=node2
欢迎使用 Iridium! 让我们提高生产力!
>>> !join_cluster 127.0.0.1 2254
尝试加入集群...
已连接到集群!
>>>
```

#### 列出成员

让我们添加一个实用命令，列出我们所知道的集群成员。首先，我们需要在 `src/cluster/manager.rs` 中添加一个函数，让我们获得别名列表：

```rust
pub fn get_client_names(&self) -> Vec<String> {
    let mut results = vec![];
    for (alias, _) in &self.clients {
        results.push(alias.to_owned());
    }
    results
}
```

在 `src/repl/mod.rs` 中，添加一个名为 `!cluster_members` 的命令。它应该调用一个函数，`cluster_members`，它是：

```rust
fn cluster_members(&mut self, args: &[&str]) {
    self.send_message(format!("列出已知节点："));
    let cluster_members = self.connection_manager.get_client_names();
    self.send_message(format!("{:#?}", cluster_members));
}
```

现在如果我们启动两个 iridium 虚拟机，加入它们，并列出成员，我们看到……一个空列表！

## 总结

这是一个停下来的好地方，因为我们需要做一些更多的更改以使所有这些连接起来。但我们现在已经有可以相互加入的节点了！

原文出处：[Clustering: Part 2 - So You Want to Build a Language VM](https://blog.subnetzero.io/post/building-language-vm-part-28/)
作者名：Fletcher

### 专有名词及注释

0. **协议 (Protocol)**：在分布式系统和网络通信中，协议是指一系列规则和标准，它们定义了数据如何在不同的节点或系统之间进行传输和交换。
1. **JSON (JavaScript Object Notation)**：
   - JSON 是一种轻量级的数据交换格式，易于人阅读和编写，同时也易于机器解析和生成。它基于 JavaScript 对象字面量，但被设计为独立于任何编程语言。
   - JSON 通常用于 Web 应用程序中的客户端与服务器之间的数据传输，以及服务器之间的数据交换。
2. **Protobuf (Protocol Buffers)**：
   - Protobuf 是由 Google 开发的一种数据序列化协议，用于通信协议、数据存储等场景。它提供了一种机制来将结构化数据序列化成二进制格式，以便于在网络中传输或者存储。
   - Protobuf 支持多种编程语言，并且可以自动生成源代码，使得序列化和反序列化变得非常方便。
3. **Flatbuf (FlatBuffers)**：
   - FlatBuffers 是 Google 开发的另一种数据序列化格式，它旨在提供一种内存效率高、访问速度快的序列化机制。
   - 与 Protobuf 不同，FlatBuffers 在序列化时不进行内存分配，这使其非常适合性能敏感的应用场景，如游戏、嵌入式设备和实时系统。
4. **自定义协议**：
   - 自定义协议是指根据特定应用的需求而设计的协议，它可能不遵循任何现成的标准或规范。
   - 自定义协议可以提供更高的灵活性和控制力，但也需要更多的开发工作和维护成本。

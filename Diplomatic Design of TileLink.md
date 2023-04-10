## Diplomatic Design of TileLink

### 概述

​	在现代SoC设计中incorporate大量专门的硬件单元，这些硬件单元必须嵌入到总线的地址空间中。这个过程有很大的工作量，并且很容易犯错。这个RISC-V模块化这个设计生产力受到处理器核参数、总线顺序、从设备等因素的影响。这些复杂性激励着新的工具还有参数化SoC设计的出现。文章中提出两个新的工具用于构建正确的共联结构。

​	Diplomacy是一个参数协商框架，除了确定系统节点之间相互的兼容性，同时通过其他节点的状态确定来例化自己。TileLink是一个高度参数化的共享内存网络标准。

### 介绍

​	这种网络结构典型地包含一个总线的拓扑结构。文章的方法是首先生成一个互联网络的图结构，然后使用这个图进行推理。Diplomacy会给出一系列互联网络节点的master节点和slave节点的表述，一个总线协议的模板。Diplomacy会交叉检查这些连接设备的属性，协商一些参数，并将这些参数绑定到adapters和endpoints来用于生成它们自己的硬件。Rocket-chip的TileLink利用Diplomacy来提供互联网络之间的各种协议一致。

![](https://raw.githubusercontent.com/PorterLu/picgo/main/rocket_chip_diplomacy_example.png)

​		Diplomacy使用两阶段硬件生成，第一个阶段进行参数协商，这一个阶段会探索图的拓扑结构，节点会协商每条边的参数。第二个阶段是具体模块的生成阶段，在这个阶段Chisel编译器根据图中的module的层次进行触发。

​		Diplomacy抽象的基础是图中的节点和边。节点使用参数生成具体的硬件，边代表了master和slave接口，这里source node表示master interface，同时sink代表了slave 接口。边最后会生成wire或者模块的IO。各个节点还会利用互联网络上对应的边的视角，来进行生成。

​	协议的参数会由节点决定或者由一些节点进行协商。Adapter节点可能会在参数上做一些变化然后传播到各个边。例如可能有如下的参数：

* 连接到一个sink节点的source节点数量。
* 连接到一个source节点的sink节点数量。
* master节点操作的type和size
* slave节点操作的type和size
* 某个地址范围上操作允许的type和size。
* 某个地址范围上其他允许的属性。
* 边上的Ordering需求（例如FIFO）
* bundle中的控制域
* bundle中数据位的宽度。

​	参数协商自己分为两个子阶段。从source endpoint nodes开始，参数从它们流向sink nodes。之后从sink endpoint nodes，参数流向连接的source nodes。因此edge可以接受到两个方向的参数。如果某一个adapter的要求没被满足，这个参数协商的过程会失败。

​	Diplomacy是总线协议独立的，任何需要参数化的协议都可以使用diplomacy。

### TILELINK

​	TileLink 是个chip-scale 互联网络标准，使得多个处理节点可以一致地访问共享内存和MMIO设备。为了防止死锁，TileLink 确定了channel之间的优先级，$A \lt B \lt C \lt D \lt E$。Channel上是有方向性的，要么是从master到slave，要么是从slave到master。

​	最为基础的两个Channel是Channel A和Channel D。

* Channel A： 在特定地址发起一次请求
* Channel D：传输data response 或者 acknowlegement

​	为了保持协议的一致性，这里添加了另外三个Channel，可以读写Cache的数据。

* Channel B：发起一次请求，这个地址已经master agent cache了.
* Channel C:  返回B通道的请求。
* Channel E：发送一次acknowlegement

![](https://raw.githubusercontent.com/PorterLu/picgo/main/diplomacy_dag.png)

### 设计

​	考虑仲裁器生成器的例子，这个仲裁器将会接受N个输入，只有一个单一的输出，所以需要在source field中添加$log_2(n)$来进行路由。Diplomacy可以从图中进行参数推断，这个参数N可以在第一阶段的参数推断出来，并且传播到接下来的过程中。

```scala
trait ConnectsToMemory {
    val processorMasterNode: TLSourceNode
    val memorySlaveNode: TLSinkNode = LazyModule(new TLRAM).node
}

trait ConnectsIncoherently extends ConnectsToMemory {
    require(processorMasterNode.master.size <= 1)
    memorySlaveNode := processorMasterNode
}

trait ConnectsViaBroadcastHub extends ConnectsToMemory {
    val hub = LazyModule(new TLBroadcastHub)
    hub.node := processorMasterNode
    memorySlaveNode := hub.node
}

trait ConnectsViaL2Cache extends ConnectsToMemory {
    val l2 = LazyModule(new TLCache)
    l2.node := processorMasterNode
    memorySlaveNode := l2.node
}

trait HasOneCore extends ConnectsToMemory {
    val core = LazyModule(new Rocket)
    val processorMasterNode = core.node
}

trait HasTwoCores extends ConnectsToMemory {
    val cores = List(2).fill(LazyModule(new Rocket))
    val xbar = LazyModule(new TLXbar)
    val processorMasterNode = xbar.node
    cores.foreach { c => xbar.node := c.node }
}

class SingleCoreSystem extends HasOneCore with ConnectsIncoherently
class DualCoreSystem extends HasTwoCores with ConnectsViaBroadcastHub
class DualCoreL2System extends HasTwoCores with ConnectsViaL2Cache

class IncompleteSystem extends ConnectsViaL2Cache
class UnsafeSystem extends ConnectsIncoherently
```

![](https://raw.githubusercontent.com/PorterLu/picgo/main/diplomacy_hasOneCore.png)
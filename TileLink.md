# TileLink and Diplomacy

## 节点类型

​	`Diplomacy`代表将`SoC`中的各个节点作为有向无环图中的节点。

### Client节点

​	`TileLink Client`是发起`TileLink`事务的节点，从`A channel`会发起请求，并且在`D channel`接受回复。如果实现了`TL-C`就可在`B channel`上接受`probe`，在`C channel`发起`release`，同时`E channel`上表示已经接受到回复。在`Rocket Chip`上`L1 Cache`和`DMA`设备可以作为`TileLink`的`Client`节点。

```scala
class MyClient(implicit p: Parameters) extends LazyModule {
    val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    	name = "my-client",
        sourceId = IdRange(0, 4),
        requestFifo = true,
        visibility = Seq(AddressSet(0x10000, 0xffff)))))))
    
    lazy val module = new LazyModule(this) {
        val (tl, edge) = node.out(0)
        //...
    }
}
```

* `name`在表示在`Diplomacy`图中，这个节点的名字。
* `SourceId`用于区分在一系列的`client`中是哪个节点。
* `requestFifo`, 表示是否根据先到先回复的顺序进行回复。
* `visibility`，表示了这个节点会访问的区域，在这个例子中为`AddressSet(0x10000, 0xffff)`, 意味着这个`client`只能访问`0x10000`到`0x1ffff`。

在`LazyModule`中使用了`tl`和`edge`：

* `tl`表示连接到这个节点的一系列`Bundle`, 可以是`TileLink`中的五个通道，可以用于节点通信。
* `edge`表示在`Diplomacy`图中的边，包含一些有用的函数。

### Manager Node

​	`TileLink manager`会从`client`处接受请求, 并且在`D Channel`上发起一次回复

```scala
class MyManager(implicit p: Parameters) extends LazyModule {
    val device = new SimpleDevice("my-device", Seq("tutorial,my-device0"))
    val beatBytes = 8
    val node = TLManagerNode(Seq(TLSlavePortParameters, v1(Seq(TLManagerParameters(
    	address = Seq(AddressSet(0x20000, 0xfff)),
        resources = device.reg,
        regionType = RegionType.UNCACHED,
        executable = true,
        supportsArithmetic = TransferSizes(1, beatBytes),
        supportsLogical = TransferSizes(1, beatBytes),
        supportsGet = TransferSizes(1, beatBytes),
        supportsPutFull = TransferSizes(1, beatBytes),
        supportsPutPartial = TransferSiszes(1, beatBytes),
        supportsHint = TransferSizes(1, beatBytes),
        fifoId = Some(0))), beatBytes)))
	lazy val module = new LazyModuleImpl(this) {
        val (tl, edge) = node.in(0)
    }
}
```

​	对于`Manager`节点需要两个参数，一个是`beatBytes`，一个是`TLManagerParameters`。其中`beatBytes`表示`TileLink`接口的宽度。接下来说明`TLManagerParameters`：

* `address`：表示了`manager`会服务的地址，在这个例子中为`0x20000`到`0x20fff`。
* `resources`, 这里我们使用了一个`SimpleDevice`, 这个参数表示表示你想在`BootRom`中的设备树中添加一个设备。
* `regionType`, 表示了这些地址的`caching`行为，比如`CACHED`、`TRACKED`、`UNCACHED`、`IDEMPOTENE`等。
* `executable`, 表示是否允许从这个节点读取指令，大多数`MMIO`设备这个选项都设置为false。
* 接下来的参数，用`support`的类型，传输的大小，是否支持`FIFO`

### Register Node

​	这种类型的节点提供了`regmap`方法，可以运行通过操作寄存器来自动完成控制`TileLink`协议。

### Identity Node

​	这种节点可以用于进行连接，例子如下：

```scala
class MyClient1(implicit p: Parameters) extends LazyModule {
    val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameter(
    "my-client1", IdRange(0, 1))))))
    
    lazy val module = new LazyModuleImpl(this) {
        //...
    }
}

class MyClient2(implicit p: Parameters) extends LazyModule {
    val node = TLClientNode(Seq(TLMasterPortParameters.v1(Seq(TLClientParameters(
    "my-client2", IdRange(0, 1))))))
    
    lazy val module = new LazyModuleImpl(this) {
        //...
    }
}
```

​	接下来我们对这两个节点进行连接：

```scala
class MyClientGroup(implicit p: Parameters) extends LazyModule {
    val client1 = LazyModule(new MyClient1)
    val client2 = LazyModule(new MyClient2)
    val node = TLIdentityNode()
    
    node := client1.node
    node := client2.node
    
    lazy val module = new LazyModuleImpl(this) {
        // Nothing to do here
    }
}
```

​	我们也可对两个`manager`节点做同样的事，我们还可以用多条边连接两个点。

```scala
class MyClientManagerComplex(implicit p: Parameters) extends LazyModule {
    val client = LazyModule(new MyClientGroup)
    val manager = LazyModule(new MyManagerGroup(8))
    
    manager.node :=* client.node
    
    lazy val module = new LazyModuleImp(this) {
        //...
    }
}
```

​	这里我们使用了`:*`符号用于进行连接，这里`client1.node`会和`manager1.node`进行连接，`client2.node`会和`manager2.node`进行连接。

### Adapter Node

​	和`Adapter Node`节点类似，`adapter`需要相同数量的输入和输出。但是和`Adapter Node`节点不同的是，它可以改变连接关系。

```scala
val node = TLAdapterNode {
    clientFn = { cp =>
        // ..
    },
    managerFn = { mp =>
        //..
    }
}
```

​	`clientFn`中我们需要传入`TLClientPortParameters`， 同时在`managerFn`我们需要传入`TLManagerPortParameters`。

### Nexus Node

​	`Nexus`节点和`adapter`节点相似，但是输入输出的接口数可以不同，这种节点主要由`TLXbar`使用，可以用于生成`TileLink`的`crossbar`。

```scala
val node = TLNexusNode {
    clientFn = { Seq =>
        //...
    },
    managerFn = { Seq => 
    	//...
    }
}
```

## 连接类型

* `:=`, 一对一的连接
* `:=*`, 进行多个连接，但是连接的边的数量由`client`决定
* `:*=`, 连接的数量由`manager`决定
* `:*=*`, 连接的数量由两边已知数量的一方决定。

## TileLink Edge Object Methods

​	我们刚才在节点类型一章中提到`(tl, edge)`，其中`edge`中有很多有用的方法。这里进行介绍`Get`

### Get

​	`Get`请求可以获取一个`BundleA`，之后可以在`D`通道收到回复，需要的参数如下

*  `fromSource`,  源`ID`
* `toAddress`，要读取的地址
* `lgSize`，需要读取的字节数

返回值`(Bool, TLBunadleA)`,  表示是否获取成功和一个`BundleA`

## Register Router

​	用于为`CPU`暴露设备，通过一组寄存器可以操作设备，对于`TileLink`设备可以通过使用`regmap`接口来扩展`TLRegisterRouter`类，或者可以创建一个`TLRegisterNode`。

```scala
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.diplomacy._
import freechips.rocketchip.regmapper._
import freechips.rocketchip.tilelink.TLRegisterNode

class MyDeviceController(implicit p: Parameters) extends LazyModule {
    val device = new SimpleDevice("my-device", Seq("tutorial,my-device0"))
    val node = TLRegisterNode(
    	address = Seq(AddressSet(0x10028000, 0xfff)),
        device = device,
        beatBytes = 8,
        concurrency = 1)
    
    lazy val module = new LazyModuleImp(this) {
        val bigReg = RegInit(0.U(64.W))
        val mediumReg = RegInit(0.U(32.W))
        val smallReg = RegInit(0.U(16.W))
        
        val tinyReg0 = RegInit(0.U(4.W))
        val tinyReg1 = RegInit(0.U(4.W))
        
        node.regmap(
        	0x00 -> Seq(RegField(64, bigReg)),
            0x08 -> Seq(RegField(32, mediumReg)),
            0x0C -> Seq(RegField(16, smallReg)),
            0x0E -> Seq(
            	RegField(4, tinyReg0),
                RegField(4, tinyReg1)
            )
        )
    }
}
```

​	上面这个例子中使用`TLRegisterNode`来映射不同大小的硬件寄存器，

* `address`， 用于说明设备的地址，也是设备寄存器的基地址
* `device`, 说明设备树的入口

* `beatBytes`用于说明接口的宽度
* `concurrency`用于说明`TlileLink`请求队列的大小。

和这个节点进行交互的方法就是调用`regmap`, 说明了偏移和写入的寄存器。

### Decoupled Interfaces

​	如果想要不仅仅是读写硬件寄存器，我们需要`RegFiled`提供对应`DecoupledIO`进行操作。如下有一个例子

```scala
val queue = Module(new Queue(UInt(64.W), 4))
node.regmap(
	0x00 -> Seq(RegField(64, queue.io.deq, queue.io.enq))
)
```

​	这里`RegField`有三个参数，第一参数指名宽度，第二个参数指明读接口，第三个参数指明写接口。这里写操作将数据压入队列，而读操作从队列中取数据。

### Using Function

​	我们还可以使用函数创建硬件寄存器

```scala
val counter = RegInit(0.U(64.W))

def readCounter(ready: Bool): (Bool, UInt) = {
    when(ready) { counter := counter - 1.U }
    (true.B, counter)
}

def writeCounter(valid:Bool, bits: UInt): Bool = {
    when(valid) { counter:= counter + 1.U }
    true.B
}

node.regmap(
	0x00 -> Seq(RegField.r(64, readCounter(_))),
    0x08 -> Seq(RegField.w(64, writeCounter(_, _)))
)
```

​	当进行读时，会传入`ready`信号，之后返回`valid`和`bits`。当进行写实时会传入`valid`信号，返回`ready`。

### 使用其他协议的设备

​	我们可以将`TLRegisterNode`转化为`AXI4Register`节点。

```scala
import freechips.rocketchip.amba.axi4.AXI4RegisterNode

class MyAXI4DeviceController(implicit p: Parameters) extends LazyModule {
    val node = AXI4RegisterNode(
    	address = AddressSet(0x10029000, 0xfff),
        beatBytes = 8,
        concurrency = 1
    )
    
    lazy val module = new LazyModuleImp(this) {
        val bigReg = RegInit(0.U(64.W))
        val mediumReg = RegInit(0.U(32.W))
        val smallReg = RegInit(0.U(16.W))
        val tinyReg0 = RegInit(0.U(4.W))
        val tinyReg1 = RegInit(0.U(4.W))
        node.regmap(
        	0x00 -> Seq(RegField(64, bigReg)),
            0x08 -> Seq(RegField(32, mediumReg)),
            0x0C -> Seq(RegField(16, smallReg)),
            0x0E -> Seq(
            	RegField(4, tinyReg0),
                RegField(4, tinyReg1),
            )
        )
    }
}
```

​	除了`AXI4`不用接受`device`参数，只能用一个`AddressSet`, 其他一样。

## Diplomatic Widgets

`TileLink Widgets`在`freechips.rocketchip.tilelink`，`AXI4 widgets`在`freechips.rocketchip.amba.axi4`中。

### TLBuffer

​	这是一个用于`buffering``TileLink`事务的工具，它是用一个队列进行实现的，

* `depth: Int`， 用于指明队列中可以容纳的数量。
* `flow: Boolean`, 用于组合`valid`信号，于是输入可以在一个周期内被处理。
* `pipe: Boolean`， 组合`ready`信号，可以队列全速运行。


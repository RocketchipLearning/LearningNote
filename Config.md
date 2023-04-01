## Config

### 预备知识

​	偏函数，在`Scala`中它是一个`Trait`,  `PartialFunction[A, B]`, 它接受一个`A`参数，返回一个`B`。

```scala
val f: PartialFunction[Int, String] = {
    case 1 => "One",
    case 2 => "Two",
    case 3 => "Three",
    case _ => "Other"
}
```

### 概述

​	在`Rocket-chip`等生成器中，可以通过配置`Config`文件来定制生成`CPU`，接下来将介绍其实现原理

### Field

```scala
package freechips.rocketchip.config

abstract class Field[T] private(val default: Option[T])
{
    def this() = this(None)
    def this(default: T) = this(Some(default))
}
```

​	我们在`config`这个子模块中定义了`Field`类，作为一个抽象类它无法实例化，但是它提供了两个方法，`this`放回一个为`None`的`Option`，而`this(default: T)`， 当以了一个内容为`default`的`Option`。

​	这个类型可以将所有的类型全部加一层`Option`。

### View

```scala
abstract class View {
    final def apply[T](pname: Field[T]): T = apply(pname, this)
    final def apply[T](pname: Field[T], site: View): T = {
        val out = find(pname, site)
        require(out.isDefined, s"Key ${pname} is not defined in Parameters")
        out.get
    }
    final def lift[T](pname: Field[T]): Option[T] = lift(pname, this)
    final def lift[T](pname: Filed[T], site: View): Option[T] = find(pname, site)															.map(_.asInstanceOf[T])
    protected[config] def find[T](pname: Field[T], site: View): Option[T]
}
```

​	这里`find`是一个抽象方法，我们需要在继承`View`后，`apply`中调用了`find`方法。

### Parameters

```scala
abstract class Parameters extends View {
  final def ++ (x: Parameters): Parameters =
    new ChainParameters(this, x)
 
  final def alter(f: (View, View, View) => PartialFunction[Any,Any]): Parameters =
    Parameters(f) ++ this
 
  final def alterPartial(f: PartialFunction[Any,Any]): Parameters =
    Parameters((_,_,_) => f) ++ this
 
  final def alterMap(m: Map[Any,Any]): Parameters =
    new MapParameters(m) ++ this
 
  protected[config] def chain[T](site: View, tail: View, pname: Field[T]): Option[T]
  protected[config] def find[T](pname: Field[T], site: View) = chain(site, new TerminalView, pname)
}
 
object Parameters {
  def empty: Parameters = new EmptyParameters
  def apply(f: (View, View, View) => PartialFunction[Any,Any]): Parameters = new PartialParameters(f)
}

private class TerminalView extends View {
  def find[T](pname: Field[T], site: View): Option[T] = pname.default
}
 
private class ChainView(head: Parameters, tail: View) extends View {
  def find[T](pname: Field[T], site: View) = head.chain(site, tail, pname)
}
```

​	`Parameters`继承了`View`，实现了`find`方法，但是方法也调用了`chain`，`chian`中的`site`表示当前视图，`tail`表示了下一试图，`pname`则是要找的配置项。

### Config

```scala
class Config(p: Parameters) extends Parameters {
    def this(f: (View, View, View)) => PartialFunction[Any, Any]) = this(Parameters(f))
    
    protected[config] def chain[T](site: View, tail: View, pname: Field[T]) = p.chain(site, tail, pname) 
    override def toString = this.getClass.getSimpleName
    def toInstance = this
}

private class ChainParameters(x: Parameters, y: Parameters) extends Parameters {
  def chain[T](site: View, tail: View, pname: Field[T]) = x.chain(site, new ChainView(y, tail), pname)
}
 
private class EmptyParameters extends Parameters {
  def chain[T](site: View, tail: View, pname: Field[T]) = tail.find(pname, site)
}
 
private class PartialParameters(f: (View, View, View) => PartialFunction[Any,Any]) extends Parameters {
  protected[config] def chain[T](site: View, tail: View, pname: Field[T]) = {
    val g = f(site, this, tail)
    if (g.isDefinedAt(pname)) Some(g.apply(pname).asInstanceOf[T]) else tail.find(pname, site)
  }
}
 
private class MapParameters(map: Map[Any, Any]) extends Parameters {
  protected[config] def chain[T](site: View, tail: View, pname: Field[T]) = {
    val g = map.get(pname)
    if (g.isDefined) Some(g.get.asInstanceOf[T]) else tail.find(pname, site)
  }
}
```

​	这里`Config`继承了`Parameters`，`chian`中直接调用了`p`参数的`chain`方法。使用`PartialParameters`，它是一个私有类。它会调用传入的隐式参数方法。

### 实现

```scala
package mini

import chisel3.Module
import freechips.rocketchip.config.{Parameters, Config}
import junctions._

class MiniConfig extends Config((site, here, up)) => {
    case XLEN => 32,
    case Trace => true,
    case BuildALU => (p: Parameters) => Module(new ALUArea()(p))
    case BuildImmGen => (p: Parameters) => Module(new ImmGenWire()(p))
    case BuildBrCond => (p: Parameters) => Module(new BrCondArea()(p))
    case NWays => 1
    case NSets => 256
    case CacheBlockBytes => 4 * (here(XLEN) >> 3) 
    case NastiKey => new NastiParameters(
    	idBits = 5，
        dataBits = 64,
        addrBits = here(XLEN))
    )
}
```

实际使用的时候，可以先看顶层：

```scala
val params = (new MiniConfig).toInstance
val chirrtl = firrtl.Parser.parse(chisel3.Driver.emit(() => new Tile(params)))
```

​	这里在`Tile`中传入了`param`参数，由于隐私参数的性质，这个参数可以不断向下传播。

```scala
class Tile(tileParams: Parameters) extends Module with TileBase {
    implicit val p = tileParams
    val io = IO(new TileIO)
    val core = Module(new Core)
    val icache = Module(new Cache)
    val dcache = Module(new Cache)
    val arb = Module(new MemArbiter)
    
    io.host <> core.io.host
    core.io.cache <> icache.io.cpu
    core.io.dcache <> dcache.io.cpu
    arb.io.icache <> icache.io.nasti
    arb.io.dcache <> dcache.io.nasti
    io.nasti <> arb.io.nasti
}
```

​	`Tile`模块把入参赋值给隐式参数`p`。之后传入接下来所有的模块。以`Core`模块为例

```scala
abstract trait CoreParams {
    implicit val p: Parameters
    val xlen = p(XLEN)
}
```

​	这里`p`我们知道传入的是一个`miniConfig`, 有一个父类类型`Parameter`进行接受是合法的。接着我们调用了`p(XLEN)`, 由于传入的`XLEN`是一个`Field`类型，这里会调用，`View`的`apply`方法，

最终会调用：

```scala
final def apply[T](pname: Field[T], site: View): T = {
    val out = find(pname, site)
    require(out.isDefined, s"Key ${pname} is not defined in Parameters")
    out.get
}
```

这里的`find`又会调用之前介绍的`PartialParameters`的`chain`方法：

```scala
protected[config] def chain[T](site: View, tail: View, pname: Field[T]) = {
	val g = f(site, this, tail)
    if (g.isDefinedAt(pname)) Some(g.apply(pname).asInstanceOf[T])
    else tail.find(pname, site)
}
```

​	之后就会跟我们的`(site, here, tail) => {case...}`进行匹配。
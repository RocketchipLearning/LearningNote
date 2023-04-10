## build.sc

```scala
import mill._
import scalalib._ 

val defaultVersions = Map(
	"chisel3" -> "3.4.3",
    "chisel3-plugin" -> "3.4.3",
    "scala" -> "2.12.13",
)
```

​	这里要使用`mill`进行编译，所以引入了`mill`模块，同时声明了一个函数，说明了使用库的版本。

```scala
def getVersion(dep: String, org: String = "edu.berkley.cs", cross: Boolean = false) = {
    val version = sys.env.getOrElse(dep + "Version", defaultVersions(dep))
    if (cross) 
    	ivy"$org:::$dep::$version"
    else
    	ivy"$org::$dep::$version"
}
```

​	这个函数使用的默认仓库位`edu.berkley.cs`, 同时交叉版本为`false`, 这里`cross`用于自动基于`scala`版本选择依赖。`ivy`在脚本中生成依赖所需要的字符串格式。

```scala
trait CommonModule extends ScalaModule {
    override def scalaVersion = defaultVersions("scala")
    override def scalacOptions = Seq("-Xsource:2.11")
    val macroParadise = ivy"org.scalamacros::paradise:2.1.1"
    override def compileIvyDeps = Agg(macroParadise)
    override def scalacPluginIvyDeps = Agg(macroParadise)
}
```

​	设置编译选项中的`-Xsource:2.11`说明要使用老版本的`scala`中的一些特性，同时了编译和插件的依赖。其中`Agg`是`Aggregate`的缩写，用于在`sbt`等脚本中对于依赖进行集成。

```scala
object `rocket-chip` extends SbtModule with CommonModule {
    override def ivyDeps = super.ivyDeps() ++ Agg(
    	ivy"${scalaOrganization()}:scala-reflect:${scalaVersion()}",
        ivy"org.json4s::json4s-jackson:3.6.1",
        getVersion("chisel3"),
    )
    
    obejct macros extends SbtModule with CommonModule
    
    object `api-config-chipsalliance` extends CommonModule {
        override def millSourcePath = super.millSourcePath ++ / "design" / "craft"
    }
    
    object hartfloat extends SbtModule with CommonModule {
        override def ivyDeps = super.ivyDeps() ++ Agg(getVersion("chisel3"))
    }
    
    object def moduleDeps = super.moduleDeps ++ Seq(
    	`api-config-chipsalliance`, macros, hardfloat
    )
}
```

​	`rocket-chip`这个工程额外依赖了三个库，第三个使用之前写的函数进行生成。

​	之后`rocket-chip`这个项目底下还有内嵌的三个`module`,  `rocket-chip`依赖这个三个`module`。

```scala
object Backend extends SbtModule with CommonModule {
    override def millSourcePath = millOuterCtx.millSourcePath
    
    override def ivyDeps = super.ivyDeps() ++ Agg(
    	getVersion("chisel3")
    )
    
    override def moduleDeps = super.moduleDeps ++ Seq(`rocket-chip`)
}
```

`millOuterCtx.millSourcePath`意思是最外层的`project`，意味着就是当前目录下的这一个，只要加上模块依赖`rocket-chip`就可以进行接下来的实验了。
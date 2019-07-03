### Java HotSpot VM Options

### Options

Java命令行支持多种选项，可以分为下面几类：

- Standard Options
- Non-Standard Options
- Advanced Runtime Options
- Advanced JIT Compiler Options
- Advanced Seriviceability Options
- Advanced Grabage Collection Options



#### Standard Options

这些是JVM的所有实现都支持的最常用选项.

**-agentlib:libname[=options]**

​	加载指定的本机代理库。在库名称之后，可以使用以逗号分隔的库特定选项列表。

​	如果`-agentlib:foo`指定了该选项，则JVM会尝试加载`libfoo.so`在`LD_LIBRARY_PATH`系统变量指定的位置命名的库（在OS X上此变量为`DYLD_LIBRARY_PATH`）。

​	以下示例现实如何加载堆性能分析工具(HPROF)，并以每20ms的间隔获取样本CPU的信息，堆栈的深度为3:

```shell
-agentlib:hprof=cpu=samples,interval=20,depth=3
```

​	以下示例说明如何加载Java调试线协议（JDWP）库并侦听端口8000上的套接字连接，在主类加载之前挂起JVM：

```shell
-agentlib:jdwp=transport=dt_socket,server=y,address=8000
```

**-agentpath:pathname[=options]**

​	加载绝对路径名指定的本机代理库。此选项等效于`-agentlib`，但使用库的完整路径和文件名。

-client

​	选择Java HotSpot客户端VM。64位版本的Java SE Development Kit（JDK）当前忽略此选项，而是使用Server JVM。

**-Dproperty=value**

​	设置系统属性，property变量是一个没有空格的字符串，表示属性的名称。value变量是表示属性值的字符串，如果value是带空格的字符串，则将其括在引号中（`-Dfoo="foo bar"`）

**-d32**

​	在32位环境中运行应用程序。如果未安装或不支持32位环境，则将报告错误。默认情况下，除非使用64位系统，否则应用程序将在32位环境中运行

**-d64**

​	在64位环境中运行应用程序。如果未安装或不支持64位环境，则将报告错误。默认情况下，除非使用64位系统，否则应用程序将在32位环境中运行。目前只有Java HotSpot Server VM支持64位操作，-server选项是隐式的，使用-d64。使用-d64忽略-client选项。这可能会在将来的版本中发生变动

】

**-disableassertions[:[packagename]…|:classname]]**

**-da[:packagename]…[:classname]**

​		关闭断言。默认选项。在所有包和类中禁用断言。

​	



#### 🔗参考

https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
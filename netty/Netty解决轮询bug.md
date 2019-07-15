## Netty 解决的BUG





### Netty解决JDK1.6NPE异常

​		在JDK1.7之前，在使用`nio`时，会出现一个NPE的空指针异常，具体原因看下面一段代码：

```java
	private static String bugLevel = null;   
	static boolean atBugLevel(String bl) {		// package-private
        if (bugLevel == null) {
            if (!sun.misc.VM.isBooted())
                return false;
            java.security.PrivilegedAction pa =
                new GetPropertyAction("sun.nio.ch.bugLevel");
// the next line can reset bugLevel to null
            bugLevel = (String)AccessController.doPrivileged(pa);
            if (bugLevel == null)
                bugLevel = "";
        }
        return (bugLevel != null) && bugLevel.equals(bl);
    }
```

上述代码，由于没有做同步处理。 当一个线程进入到返回语句的**(bugLevel != null)**时，此时bugLevel不为null，但是当另外一个线程执行**AccessController.doPrivileged(pa)**时，此时会获取bugLevel的初始值`null`，此时第一个线程调用**bugLevel.equals(bl)**，bugLevel可能为空，即会出现NPE。

​		**注意⚠️：** 这个bug在**jdk7**中已经被修复，在**JDK1.8**中的代码如下所示：

```java
private static volatile String bugLevel;
static boolean atBugLevel(String var0) {
    if (bugLevel == null) {
      if (!VM.isBooted()) {
        return false;
      }

      String var1 = (String)AccessController.doPrivileged(new GetPropertyAction("sun.nio.ch.bugLevel"));
      bugLevel = var1 != null ? var1 : "";
    }

    return bugLevel.equals(var0);
  }
```

针对并发的情况，使用了`volatile`，关键字保证多线程情况下的变量值的可见性，使用`local val1`来避免多线程内存共享的，局部变量只存在各自的线程栈中，属于线程安全的。针对`bugLevel`的负值操作，采用三元运算符，此时不管怎样`bugLevel`的值都不可能为`null`。



**且看Netty是如何解决的**

**io.netty.channel.nio.NioEventLoop**

```java
 static {
        final String key = "sun.nio.ch.bugLevel";
        final String bugLevel = SystemPropertyUtil.get(key);
        if (bugLevel == null) {
            try {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    @Override
                    public Void run() {
                        System.setProperty(key, "");
                        return null;
                    }
                });
            } catch (final SecurityException e) {
                logger.debug("Unable to get/set System Property: " + key, e);
            }
        }

        int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
        if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
            selectorAutoRebuildThreshold = 0;
        }

        SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.noKeySetOptimization: {}", DISABLE_KEY_SET_OPTIMIZATION);
            logger.debug("-Dio.netty.selectorAutoRebuildThreshold: {}", SELECTOR_AUTO_REBUILD_THRESHOLD);
        }
    }
```

在类加载（天然的线程安全）的时候，使用使用静态代码块，设置系统属性`sun.nio.ch.bugLevel=""`。










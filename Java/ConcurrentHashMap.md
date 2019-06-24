## ConcurrentHashMap解读







##### 计算size大小

​	`ConcurrentHashMap`在内部采用两种方式来实现内部元素的计数。

- `baseCount`  基础计数值
- `CounterCell` 计数单元格

**baseCount**

这只是一个基础的计数变量，主要用于没有并发冲突时，在表初始化时，如果存在竞争关系，则会作为一种降级策略使用。实现机制是通过`Unsafe`的`CAS`来更新值。

**CounterCell**

`CounterCell` 字面意思可以称之为计数单元格，`ConcurrentHashMap`在内部维护多个`CounterCell`来实现内部元素的计数功能，`CounterCell`是一个`LongAdder`和`Striped64`的改编实现。通过在内部维持一个`volatitle`类型的`long`类型变量，实现了多线程的内存的可见性。其特点是：

- 多个`CounterCell`分别计数，可以减少并发冲突
- 延迟计算

- 懒加载特性，只会被动的动态扩容（2的n次方）

相关源码：

```java
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
    	
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

`addCount`每次`put`操作都要被调用。可以看出来`baseCount`仅仅只是一个后备计数方案，每次计数，优先考虑的还是`CounterCell`。这个方法大致分为如下几步：

- 
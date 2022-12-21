# Thread.VolatileRead() vs Volatile.Read()
取自 [stackoverflow](https://stackoverflow.com/questions/22569274/thread-volatileread-vs-volatile-read) 的最高分

- 为什么要写这个？？
  - 因为看的云里雾里。
- 我为什么会看到这个？？

## 正文

两个概念：

> **acquire-fence**: A memory barrier in which other reads and writes are not allowed to move before the fence.  
> **release-fence**: A memory barrier in which other reads and writes are not allowed to move after the fence.

获得栅栏时: memory barrier之后的读取或写入操作不允许在 memory barrier 之前  
释放栅栏时：memory barrier之前的读取和写入操作不允许在 memory barrier 之后

这里指的是应该时读写真正的内存

用 ↑ 表示释放栅栏，用 ↓ 表示获得栅栏。

### Volatile.Read

```c#
// Example using Volatile.Read
x = 13;
var local = y; // Volatile.Read
↓              // acquire-fence
z = 13;
```

可能发生的顺序：
```
Volatile.Read
---------------------------------------
write x    |    read y     |    read y
read y     |    write x    |    write z
write z    |    write z    |    write x
```

### Thread.VolatileRead

```c#
// Example using Thread.VolatileRead
x = 13;
var local = y; // inside Thread.VolatileRead
↑              // Thread.MemoryBarrier / release-fence
↓              // Thread.MemoryBarrier / acquire-fence
z = 13;
```

可能发生的顺序：
```
Thread.VolatileRead
-----------------------
write x    |    read y
read y     |    write x
write z    |    write z
```

大概意思就是 确保了MemoryBarrier前后代码的顺序... but...看了源码之后... 

```c#
// Volatile.Read
public static int Read(ref int location)
{
    // 
    // The VM will replace this with a more efficient implementation.
    //
    var value = location;
    Thread.MemoryBarrier();
    return value;
}
```
```c#
// Thread.VolatileRead
public static int VolatileRead(ref int address)
{
    int ret = address;
    MemoryBarrier(); // Call MemoryBarrier to ensure the proper semantic in a portable way.
    return ret;
}
```

嗯... 是一样的。于是又回去找答案 发现了一段话

> Of course it has been claimed that Microsoft's implementation of the CLI (the .NET Framework) and the x86 hardware already guarantee release-fence semantics for all writes. So in that case there may not be any difference between the two calls. On an ARM processor with Mono? Things might be different in that case.

所以说，在CLI 和 x86硬件的平台他们的是一样的。但在Arm等其他平台 他们可能会不一样。

在MSDN上对于Thread.VolatileRead和Volatile.Read有一个解释：

> Thread.VolatileRead and Thread.VolatileWrite are legacy APIs and have been replaced by Volatile.Read and Volatile.Write. See the Volatile class for more information.

说的是Volatile.Read 是Thread.VolatileRead的替代品。我只能说他的例子是和msdn的说法是矛盾的。

MSDN对于 Volatile 有更易懂的[说明](https://learn.microsoft.com/en-us/dotnet/api/system.threading.volatile?view=net-7.0&viewFallbackFrom=netframework-4.0)：

> On a multiprocessor system, due to performance optimizations in the compiler or processor, regular memory operations may appear to be reordered when multiple processors are operating on the same memory. Volatile memory operations prevent certain types of reordering with respect to the operation. A volatile write operation prevents earlier memory operations on the thread from being reordered to occur after the volatile write. A volatile read operation prevents later memory operations on the thread from being reordered to occur before the volatile read. These operations might involve memory barriers on some processors, which can affect performance.

由于编译器和性能优化问题，在多处理器系统上，会存在多处理器在同一内存块上执行的情况，而常规的内存操作可能会被重新排序。Volatile的作用在于阻止相关操作的重新排序。

|Thread 1 |	Thread 2 |
|:---:|:---:|
|x = 1;	|int y2 = Volatile.Read(ref y);|
|Volatile.Write(ref y, 1); |	int x2 = x;|

x, y 初始化都为0， 保证了如果: y2 = 1; 那么 x = 1 是肯定的，而不会发生 y2 = 1, x = 0; 那也意味着如果 y2 = 0;那么x2 = 0; 不会发生y2 = 0，x2 = 1;

原文是：
> The volatile read and write prevent reordering of the two operations within each thread, such as by the compiler or by the processor. Regardless of the order in which these operations actually occur on one thread relative to the other thread, even on a multiprocessor system where the threads may run on different processors, the volatile operations guarantee that thread 2 would not see y2 == 1 and x2 == 0. On thread 1 the write to x must appear to occur before the volatile write to y, and on thread 2 the read of x must appear to occur after the volatile read of y. So if thread 2 sees y2 == 1, it must also see x2 == 1.

这个可以理解， 但是接下来就蒙了：

相同情况下并确定了顺序：

|Sequence|Thread 1 |	Thread 2 |
|:---:|:---:|:---:|
|1|x = 1;	|...|
|2|Volatile.Write(ref y, 1); |...|
|3|...|int y2 = Volatile.Read(ref y);|
|4|...|int x2 = x;|

原文：
> Even though the volatile write to y on thread 1 occurred before the volatile read of y on thread 2, thread 2 may still see y2 == 0. The volatile write to y does not guarantee that a following volatile read of y on a different processor will see the updated value.

这里又说 在不同处理器上读取 y2 有可能是0 ... 前面明明说是即使在不同处理器上...

~所以不同处理器到底是咋样，，我没理解错吧...~

果然还是我理解的有问题：

前面说的是 y2=1, x2=1 or y2=0,x2=0 这是保证的

这里说的是在 Thread 1 先执行的情况下,  Thread2中 y2 = 0； 不能保证的是在不同处理器上读到是更新后的值。

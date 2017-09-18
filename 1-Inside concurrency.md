# 并发编程 cookbook
##内存栅栏
**概览**
内存栅栏是为了在乱序执行的处理器上保证执行顺序及内存可见性的手段。
四种细粒度的内存栅栏
1. loadload
     load1 #loadload load2	:	load1 happen-before load2
2. storestore
    store1 #storestore store2	:	store1 happen-before store2
3. loadstore
    store1 #storestore store2	:	store1 happen-before store2
4. storeload
     store1 #storeload load1	:	store1 happen-before load1
    
通过配对使用内存栅栏可以实现同步. 举例说明各种栅栏的作用

- **#loadload** 和 **#storestore**

    A 和 B 默认为 0
     
| CPU1          | CPU2          |
| :------------ | :-------------|
| A=1;          | Y=B;          |
| #storestore   | #loadload     |
| B=1;          | X=A;          |

对 CPU2 的执行结果检查，如果 Y=1 则必有 X=1。但如果 Y=0 则 X 的值可能为 0 或 1. 这很像一种“半同步”关系：并非任意时刻该同步关系都满足。

- **#loadstore**

    A 和 B 默认为 0
    
| CPU1          | CPU2          |
| :------------ | :-------------|
| X=A;          | Y=B;          |
| #loadstore    | #loadstore    |
| B=1;          | A=1;          |
检查 CPU1 和 CPU2 执行后的结果，若 Y=1，则 X=0；若 X=1，则 Y=0；

- **#loadstore**  和 **#storestore**

    A 和 B 默认为 0
    
| CPU1          | CPU2          |
| :------------ | :-------------|
| X=A;          | B=2;          |
| #loadstore    | #storestore   |
| B=1;          | A=1;          |
检查 CPU1 和 CPU2 执行后的结果，若 X=1 则 B=1 (B用新值覆盖旧值)；若 X=0，则B为0或1。

- **有点特殊的 #storeload**

    A 和 B 默认为 0
    
| CPU1          | CPU2          |
| :------------ | :-------------|
| B=1;          | B=2;          |
| #storeload    | #storeload    |
| X=B;          |               |

 检查 CPU1 执行结果，X 可能为 1 或 2。在松散型内存模型 CPU 上，X 很有可能直接从 store-buffer 中取出上一步 B 写入的 1 (***写操作对于随后的读操作存在边际效应***)，虽然可能  CPU2 已经执行了 B=2. 使用了某些实现 #storeload 语义的指令后，CPU2 写入 B=2 之后(直接写入主存)会向其它 CPU 发送 Invalidate Queue 消息，告之各个 CPU 中的 B 的缓存是失效状态，需要从主存中出 B。而 CPU1 正要执行 #storeload 对应的指令，那么它会处理 Invalidate Queue 消息(可以是直接从 cache 中标记 B 是无效缓存). 之后再执行 X=B 就可以加载到正确的值了。
**#storeload** 特殊就特殊在：
- 它在背后做了很多事情以维持内存可见性
- 耗费一般来说远高于其它三种内存栅栏
 PowerPC 使用 __lwsync 来实现 #loadload, #loadstore, #storestore 这三种栅栏，使用更重量级的 sync 来实现 #storeload.
最后，务必注意，内存栅栏可以用来实现同步，但它不是同步原语。


##acquire, release 语义
原始定义
<blockquote>An operation with acquire semantics is one which does not permit subsequent memory operations to be advanced before it. Conversely, an operation with release semantics is one which does not permit preceding memory operations to be delayed past it.</blockquote>
这两个语义的定义规定了代码的执行顺序和内存可见性. 满足这两个语义的设计也就满足了 happen-before 关系.
[等价的更细致的定义][1]
<blockquote>Acquire semantics is a property that can only apply to operations that read from shared memory, whether they are read-modify-write operations or plain loads. The operation is then considered a read-acquire. Acquire semantics prevent memory reordering of the read-acquire with any read or write operation that follows it in program order.  </blockquote>
<blockquote>Release semantics is a property that can only apply to operations that write to shared memory, whether they are read-modify-write operations or plain stores. The operation is then considered a write-release. Release semantics prevent memory reordering of the write-release with any read or write operation that precedes it in program order.</blockquote>
也即:
- acquire 语义规定了在 read 共享变量后面的操作必须发生在 read 操作之后, (read happen-before)
- release 语义规定了在 write 共享变量前面的操作必须发生在 write 操作之前, (happen-before write)

结合这两个语义可以实现 syncronizes-with 关系.
有一个有意思的问题, [Can Reordering of Release/Acquire Operations Introduce Deadlock?][2]
在上面那篇文章中，作者给出一个例子：
thread1 执行以下代码：

```
// Lock A
int expected = 0;
while (!A.compare_exchange_weak(expected, 1, std::memory_order_acquire)) {
    expected = 0;
}

// Unlock A
A.store(0, std::memory_order_release);	//【关键行】：此行能否和后面的 while 执行乱序

// Lock B
while (!B.compare_exchange_weak(expected, 1, std::memory_order_acquire)) {
    expected = 0;
}
 
// Unlock B
B.store(0, std::memory_order_release);
```
thread2 执行以下代码:

```
// Lock B
int expected = 0;
while (!B.compare_exchange_weak(expected, 1, std::memory_order_acquire)) {
    expected = 0;
}

// Lock A
while (!A.compare_exchange_weak(expected, 1, std::memory_order_acquire)) {
    expected = 0;
}

// Unlock A
A.store(0, std::memory_order_release);                      

// Unlock B
B.store(0, std::memory_order_release);
```

thread1 中标注为 **关键行** 的那行代码能否与它下面那个 while 乱序执行呢? 如果能乱序，将导致死锁。该作者认为不会乱序，因为 C++11 的内存模型保证了原子操作的内存可见性:
> An implementation should ensure that the last value (in modification order) assigned by an atomic or synchronization operation will become visible to all other threads in a finite period of time.

假如**关键行** 与 while 乱序执行, 这就有可能因为 while 是一个死循环导致与上面的内存模型保证相违背。因此该作者认为不会乱序.
实事上确实也不会发生乱序执行，但我觉得这可能不是本质原因。C++11 提供的原子操作允许我们在弱内存模型上灵活地控制happen 顺序(可见性顺序, 不仅仅是执行顺序)，这同时也意味着，原子操作也是可以被乱序执行的，这与 C++11 的内存模型规定并不矛盾。
再看这两句代码
```
A.store(0, std::memory_order_release); //在此句代码之前的代码的读或写都发生在此之前
while (!B.compare_exchange_weak(expected, 1, std::memory_order_acquire))//之后的读写发生在之后
```
第一句具有 release 语义，第二句具有 acquire 语义。相应的内存可见性限制以注释形式加在代码之后。从语义上讲，这两句代码确实可以乱序！我们看看这两个语义是怎么实现的：

![](/1/acq-rel-barriers.png)

```
acquire 语义
	load1
	#loadload    //#loadload 栅栏保证了 load2 决不会 happen-before load1
	load2 		

	load1
	#loadstore  //#loadstore 栅栏保证了 store2 决不会 happen-before load1  
	store2		
#loadload | #loadstore 结合实现了 acquire 语义:所有在 acquire 之后的语句发生于 acquire 之前的语句之后

release 语义
	store1
	#storestore    //#storestore 栅栏保证了 store2 决不会 happen-before store1
	store2		

	load1
	#loadstore    //#loadstore 栅栏保证了 store2 决不会 happen-before load1
	store2		
#loadstore | #storestore 结合实现了 release 语义:所有在 release 之前的语句发生于 release 之后的语句之前
```

由于 acquire 语义使用 #loadload 结合 #loadstore 实现，release 语义使用 #loadstore 结合 #storestore 实现，那么 acquire 和 release 比其语义做了更多的事。
> Once you digest the above definitions, it’s not hard to see that acquire and release semantics can be achieved using simple combinations of the memory barrier types I described at length in my previous post. The barriers must (somehow) be placed after the read-acquire operation, but before the write-release. [Update: Please note that these barriers are technically more strict than what’s required for acquire and release semantics on a single memory operation, but they do achieve the desired effect.]

回到上面的问题，很显示, 由于实现原因，release 语义后面的 acquire 语义不会乱序.


##syncronizes-with 关系
**概览**
- 是一种运行时的同步关系. 并非是源代码中的关系
- 可以说，内存栅栏的存在完全全是为了构建 happen-before 关系, 而 syncronizes-with 关系又是实现 happen-before 关系的一个
方法.
- syncronizes-with 关系必然涉及到两个变量: guard variable 和 payload. 也即守护变量和真正使用的变量.

编写下面的c++11 代码构建 ***写guard variable*** 与 ***读 payload*** 的 happen-before 关系.
 
``` 
void SendTestMessage(void* param)
{
    // Copy to shared memory using non-atomic stores.
    g_payload.tick  = clock();
    g_payload.str   = "TestMessage";
    g_payload.param = param;
    
    // Perform an atomic write-release to indicate that the message is ready.
    g_guard.store(1, std::memory_order_release);
}

bool TryReceiveMessage(Message& result)
{
    // Perform an atomic read-acquire to check whether the message is ready.
    int ready = g_guard.load(std::memory_order_acquire);
    
    if (ready != 0)
    {
        // Yes. Copy from shared memory using non-atomic loads.
        result.tick  = g_payload.tick;
        result.str   = g_payload.str;
        result.param = g_payload.param;
        
        return true;
    }
    
    // No.
    return false;
}
```

- 注意这种 syncronizes-with 关系是运行时关系, 并非是源代码中的关系. TryReceiveMessage 这个函数名前缀的 try
充分地说明问题, TryReceiveMessage 时，不一定有 1==ready, 然而只要有 1==ready 即表示 result 数据已经
准备好了可以接收消息. 是否构成 happen-before 关系取决于运行时的状态. 比如，我们可以一直循环执行
TryReceiveMessage 直到 1==ready, 这是不是有点自旋锁的意味?
- 还需要注意的一点是，对 payload 变量的 load 或 store 可以不是原子的. 如下图所示, acquire 语义阻止了所有在它后面的读或写重排序. release 语义阻止了所有在它前面的读或写重排序

![](/1/two-cones.png)

 **(再次强调: 类似于内存栅栏需要成对使用, acquire 和 release 成对使用才能构成 syncronizes-with 关系.)**
即使 g_payload 的读/写不是原子的, acquire/release 语义也保证了当 1==g_guard 被观察到时 g_payload 的所
有读/写必然已经完成. 
- 再研究一个问题:  g_guard 的读/写能否可以不是原子的呢? 答案是不能. 简化上面的代码如下:

```

//thread1
store g_payload
#store release
store g_guard	//该行代码后没有任何语义保证 store g_guard 什么时候执行完, 也就无法保证可见的时间

//thread2
load g_guard	//如上所说此处 g_guard 可能是部分可见.
#acquire load
load g_payload

```

&ensp;&ensp;&ensp;&ensp;综上, 对 g_guard 的load&load 应该是原子的.


acquire 和 release 的存在就是为了实现一种 syncronizes-with 关系, 进而实现 happen-before 关系.
而各种内存栅栏又实现了 acquire 和 release 语义(虽然内存栅栏提供的功能比这两个语义更严格, 更多的功能).

一些有用的参考文献:

[Preshing on Programming][http://preshing.com/]

[并发编程网 The Jsr-133 cookbook for Compiler Writers 译文][http://ifeve.com/jmm-faq/]

[CSDN java 并发编程专栏][http://blog.csdn.net/column/details/concurrency.html]

[Cpp Concurrency In Action][https://github.com/xiaoweiChen/Cpp_Concurrency_In_Action]

[linux kernel document][https://www.kernel.org/doc/Documentation/memory-barriers.txt]


作者：eddgarlli
博客：http://blog.csdn.net/lizhihaoweiwei
來源：CSDN
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



 [1]: http://maxiang.info/client_zh
 [2]: http://preshing.com/20170612/can-reordering-of-release-acquire-operations-introduce-deadlock/ 
 [3]: https://github.com/juniorfans/blog/raw/master/1/acq-rel-barriers.png
 [4]: https://github.com/juniorfans/blog/raw/master/1/two-cones.png
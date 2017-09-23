## condvar 的实现

# 概述

有一种较复杂的线程同步模型，当一个或多个线程需要等到某个条件满足，才能继续往后执行。另外的一个或多个线程可以更改那个条件，使其满足并触发。这一线程模型有很多用处，如多生产者-多消费者模型，读写锁等。我们来研究一下它可能的实现方法。

# 轮询

最次的可能是轮循等待，如下：

```
while(!condvar)
{
    //do something
}
```

轮询的方法太浪费 CPU 时间片，如果我们在等待的时候不占用 CPU 时间就好了。那也就意味着，当条件不满足时，当前线程应该放弃 CPU，直到条件满足能立即恢复往后执行。  
**上面一句话需要从两个层面看：**

* `当条件不满足时，当前线程放弃 CPU`： 这意味着需要通过一个系统调用由用户模式陷入内核模式，操作系统的线程调度器将该线程从调度队列中暂时移除且保存其线程上下文。
* `直到条件满足立即恢复往后执行`：这意味着条件满足时，\(另一个线程，即触发线程\)需要通知内核，由内核将线程放回调度队列。

线程变量就是为了解决这个问题的。我们现在要探讨线程变量的设计与实现。

# 初探

最基本地，提供一个函数来实现等待：

```
void waitCond(Condvar &cond)
{
    switchToCoreAndWait(cond); //放弃时间片调度，进入内核模式等待
}
```

相应地，另外一个线程应该在合适的时机，先使条件满足，再调用一个 signalCond 通知等待的线程

```
void signalCond(Condvar &cond)
{
    signalToWake(cond); //唤醒一个等待在 cond 上的线程
}
```

问题看起来解决了，但，这是全部吗？答案是否定的。
我们需要保证 `waitCond` 和 `signalCond` 的调用顺序和可期待的结果\(不与调用者的行为发生客观性的矛盾\)。  
我们无法知道哪一个在前，哪一个在后：若 `sinalCond` 在前执行，`waitCond` 在后，则 `waitCond` 是一个无穷等待。  
也许你会辩称，在逻辑上，先调用 waitCond 不就可以了吗。但是这是一个一厢情愿的想法，如下的代码

```
//thread1
waitCond(g_cond);


//thread2
xxxx
yyy
zzz
...
signalCond(g_cond);
```

问题的关键在于，即使 thread1 只有一行 waitCond 代码，而 thread2 有很多行代码并且 signalCond 在最后一行，在多核处理器上，thread1 先于 thread2 运行，能保证 waitCond 中的 `switchToCoreAndWait` 函数一定先于 signalCond 执行吗？必然不能！因为现代处理器允许超标量流水线处理，这存在着大量指令重排序，因此，在没有同步干涉的情况下不能预计两个不相关线程的执行先后顺序。  
上面的结论会导致`违反直觉`的行为发生：程序员的调用顺序上，waitCond 先于 signalCond，但 waitCond 却等不到后面 signalCond 的触发/唤醒！   
因此需要做一些同步来保证逻辑上的正确性。

# 再探

我们需要同步！  
尝试优化为下面的形式：

```
//thread1:
lock(g_mutex);
waitCond(g_cond);
unlock(g_mutex);

//thread2
lock(g_mutex);
xxxx
yyy
zzz
....
signalcond(g_cond);
unlock(g_mutex);
```

和上一个版本不同的是，只要 thread1 先抢到锁，就一定可以保证 waitCond 先执行。这就保证了得到的行为与程序员的设计是一致地！  
但这里有两个陷阱：  
1.如果按设想的来，thread1 先抢到锁，进入等待。十分隐蔽的一个错误是：thread2 永远无法获得锁，也就不会有机会调用 signalCond。这会导致 thread1 和 thread2 死锁！  
2.如果 thread2 先执行呢？那岂不是会造成 signalCond 先于 waitCond 执行？而这会造成后者死等待！  
下面我们一个一个解决这些问题。

# 深入

解决的思路是比较简单的：在线程真正开始等待之前应该释放锁。那是不是应该如下这样呢？

```
//thread1:
lock(g_mutex);
unlock(g_mutex);
waitCond(g_cond);
unlock(g_mutex);
```

上面的代码看起来有点奇怪： `lock` 后立即 `unlock`.  
我们可以将 unlock 放到 waitCond 内部：

```
void waitCond(Condvar &cond, Mutex &mutex)
{
    unlock(mutex);
    switchToCoreAndWait(cond); //放弃时间片调度，进入内核模式等待
}
```

等等，似乎还有问题：最后一行的 unlock 没有对应的 lock ！这会引起什么问题呢？假设 thread1 有这样的代码：

```
//thread1:
lock(g_mutex);
waitCond(g_cond, g_mutex);
//do something after cond satisfied
xxx
yyy
zzz
unlock(g_mutex);
```

在条件变量满足后， waitCond 返回，做这个线程想做的事，注意，因为 xxx, yyy, zzz 这几行代码被包围在 g\_mutex 的锁定区，调用者的意图是，这几行代码可能存在竞争，需要安全地执行。反观我们的设计 waitCond 函数，switchToCoreAndWait 返回后，往下执行。注意此时 thread1 已经失去了锁的保护，不再是线程安全的了。于是，后面的 xxx, yyy, zzz 会面临竞争的危险。另外一个问题是，由于锁已经被释放，此时，调用者写的那行 unlock 可能导致未知的行为，这取决于 Mutex 的设计。  
解决的方法其实也很简单，库函数 waitCond 既然释放了锁，那也就得需要它再锁上，同时，我们把 signalCond 代码也放下来，构成这两个库函数的实现：

```
void waitCond(Condvar &cond, Mutex &mutex)
{
    unlock(mutex);
    switchToCoreAndWait(cond); //放弃时间片调度，进入内核模式等待
    lock(mutex);
}
void signalCond(Condvar &cond)
{
    signalToWake(cond); //唤醒一个等待在 cond 上的线程
}
```

这样既完美与调用者代码融合\(lock 与 unlock 配对\)，也保证了逻辑正确性。需要注意的是，***waitCond 现在多了一个参数：mutex***：它是由调用得传入的锁，因此，被称为用户锁。不管是 glibc 实现的条件变量的 waitCond 函数，还是 Win32 的 waitCond，都有这个用户锁变量参数，而很多人往往没有把握到这个本质。
便于阅读，我们把这两个函数的使用也列出：

```
//thread1:
lock(g_mutex);
waitCond(g_cond);
unlock(g_mutex);

//thread2
lock(g_mutex);
signalCond(g_cond);
unlock(g_mutex);
```

我们已经到了这儿，所有的问题都被解决了吗 ？我们注意到， `waitCond` 中的两行代码：`unlock` 和 `switchToCoreAndWait`：先释放锁再进行等待。这会不会有一个问题：线程1获得锁后 waitCond 先被调用，执行 unlock 释放锁，不幸地是，线程1的时间片被剥夺，所在的 CPU 去执行其它线程了。而另一个 CPU 上的线程2获得了锁，执行 signalCond 作唤醒动作。  
看出问题了吗：waitCond 先执行，但是让后执行的 signalCond 扑了个空\(程序的意图可能就是想让这个 signal 去唤醒\)！多么令人惋惜的追求！   
这引出了条件变量中一个非常精华的问题：保证释放锁与进入等待是原子的。

# 更进一步

先聊一下原子。狭义的原子即是，一个操作/一个指令，可以无间隙无停顿地执行完，观察者观测的结果要么为空要么为全部。广义的原子是逻辑上/总体上/结果上的原子：要么多个线程不会同时运行，要么同时运行相互之间没有副作用，各自做的操作序列可以认为是无间隙地运行。  
举例说明，以下两个线程各自在锁保护的范围内做的操作是原子的，因为任意时刻，只会有一个线程在运行。

```
//threada
lock(mutex);
if(0 == x)
{
x += 100;
y -= 9;
}
unlock(mutex);

//threadb
lock(mutex);
if(0 > y)
{ 
x -= 50;
y += 18;
}
unlock(mutex);
```

很显然，`unlock` 和 `switchToCoreAndWait` 是两个操作，不可能实现为一条原子指令，因此，我们只能考虑逻辑上的原子。  
能不能使用常规的做法，将可能有副作用的多个线程同步，强制顺序执行呢？即：  
```
lock(__inner_mutex);
unlock(mutex);
switchToCoreAndWait(cond); //放弃时间片调度，进入内核模式等待
unlock(__inner_mutex);
```

看起来 work，但考虑这几行代码运行的环境，这将引发之前遇到过的问题：本线程加锁之后等待，没有其他线程释放锁。这个就相当于**将钥匙投进了上了锁的个人信箱里面**。  
这个法子不行，我们再 review 我们的需求：`unlock` 和 `switchToCoreAndWait` 原子化，更深层次的原因是防止前面的 waiter 得不到 signal\(先调用 waitCond，再调用 signalCond\)。  
由于 waitCond 或 signalCond 被调用时，一定已经拿到锁，因此，锁实现了 waitCond 与 signalCond 之间的 happen-before 关系：先执行的 waitCond 带来的影响必定可以被 signalCond 观察到。因此，我们可以在 waitCond 释放锁之前将自己标记为一个等待者，这样 signalCond 在获得锁后就可以看到这个等待者，在触发时，就不会跳过这个等待者。  
下面给最新版 glibc 关于这一点的实现：

1. wait 里面先增加了 \_\_wseq\(waiter sequence\)和 waiter reference，达到注册“等待者”的目的，再释放锁。于是在 signal 中可以观察到等待者。
2. 每个 waiter 增加 \_\_wseq 后得到的值即是这个 waiter 在等待队列中的位置。
3. wait 中使用的 futex\_wait 关键字与 signal 中 futex\_wake 是一样的关键字: \_\_g\_signals。\_\_g\_signals 大于0表示可唤醒等待者。futex\_wait 期望该关键字值为0，futex\_wake 设置该关键字值为1.
4. 使用两组机制：G1 组由那些有机会消耗 signals 的 waiters 组成。新到来的 signals 将一直唤醒该组的 waiters 直到 G1 所有的 waiters 都被唤醒。G2 组由后到达的 waiters 组成\(此时 G1 中还存在未被唤醒的 waiters\)。当 G1 中所有的 waiters 都被唤醒且有一个新的 signal 到达，则这个 signal 将 G2 转化为新的 G1
5. wait 中释放锁后，自旋等待，检查 \_\_g\_signals，自旋次数结束，进入 futex\_wait。（省掉掉了异常处理：惊群效应，取消等待，组关闭等）
6. signal 中获得锁后，检查 \_\_wseq，当没有等待者直接返回。否则获得锁，检查是否需要切换组\(例如首次调用 wait 后 G1 为空，G2有一个等待者，则首次调用 signal 后需要将 G2 切换为 G1\)，递增 \_\_g\_signals，递减 \_\_g\_size\(未唤醒的 waiters 个数\)，再调用 futex\_wake。
7. 由5和6可知，若线程A wait，线程B signal，有以下破坏“释放锁并等待”的执行顺序：“A-释放锁，B-获得锁，B-递增 \_\_g\_signals，B-futex\_wake，A-futex\_wait”，该执行顺序下，最后一个 A-futex\_wait由于 futex\_wait 期望的关键字 \_\_g\_signals 值不为 0 则它不会进入等待，被直接唤醒；若执行的顺序是，“A-释放锁，B-获得锁，B-递增 \_\_g\_signals，A-futex\_wait，B-futex\_wake”，由原因同上，futex\_wait 并不会真正进入等待。这两种情况下的 futex\_wake 没有任何作用\(它本来不会引起阻塞，调用无害\)！

需要注意的是，上面的 G1, G2本身其实与`释放锁-等待`原子化无关，它主要是用于解决 Releax-MO 中的 ABA 问题的。
#结语
本文从最基本的想法开始，一步一步探讨了条件变量的设计与实现。实际上，还有很多的细节需要考虑：效率，ABA问题，内存可见性问题，等待，都需要殚精竭虑！这些本文都没有涉及，有兴趣深入地，推荐使用在线C++源码阅读站看看 glibc 这块的源码：
[glibc-pthread_cond_wait][https://code.woboq.org/userspace/glibc/nptl/pthread_cond_wait.c.html]
[glibc-pthread_cond_signal][https://code.woboq.org/userspace/glibc/nptl/pthread_cond_signal.c.html]


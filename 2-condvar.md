##condvar 的实现
#概述
有一种较复杂的线程同步模型，当一个或多个线程需要等到某个条件满足，才能继续往后执行。另外的一个或多个线程可以更改那个条件，使其满足。这一线程模型是一个固定存在的模型，我们来研究一下，有哪些可能的实现方法。

#轮询
最次的可能是轮循等待，如下：

```
while(!condvar)
{
    //do something
}
```
轮询的方法太浪费 CPU 时间片，如果我们在等待的时候不占用 CPU 时间就好了。那也就意味着，当条件不满足时，当前线程应该放弃 CPU，直到条件满足能立即恢复往后执行(操作系统将线程放回到线程调度队列)。
**上面一句话需要从两个层面看：**
- `当条件不满足时，当前线程放弃 CPU`： 这意味着需要通过一个系统调用由用户模式陷入内核模式，操作系统的线程调度器将该线程从调度队列中暂时移除且内核保存线程上下文。
- `直到条件满足立即恢复往后执行`：这意味着条件满足时，(另一个线程，即触发线程)需要通知内核，由内核将线程放回调度队列。

线程变量就是为了解决这个问题的。我们现在要探讨线程变量的设计与实现。

#初探
最基本地，我们我们可以对外提供一个函数来实现等待，对应地：

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

问题看起来解决了，但，这是全部吗？答案是否定的
我们需要保证 `waitCond` 和 `signalCond` 的调用顺序和可期待的结果(不与调用者的行为发生客观性的矛盾)。
按现在提供的函数，无法知道哪一个在前，哪一个在的，若 `sinalCond` 在前执行，`waitCond` 在后，则 `waitCond` 是一个无穷等待。
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
#再探
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

和上一个版本不同的是，只要 thread1 先抢到锁，就一定可以保证 waitCond 先执行。这就保证了行为与程序员的设计是一致地！
但这里有两个陷阱：
1.如果按设想的来，thread1  先抢到锁，进入等待。但十分隐蔽的一个错误是：thread2 永远无法获得锁，也就不会有机会调用 signalCond。这会导致 thread1 和 thread2 死锁！
2.如果 thread2 先执行呢？那岂不是会造成 signalCond 先于 waitCond 执行？而这会造成后者死等待！
下面我们一个一个解决这些问题。

#深入
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
在条件变量满足后， waitCond 返回，做这个线程想做的事，注意，因为 xxx, yyy, zzz 这几行代码被包围在 g_mutex 的锁定区，调用者的意图是，这几行代码可能存在竞争，需要安全地执行。反观我们的设计 waitCond 函数，switchToCoreAndWait 返回后，往下执行。注意此时 thread1 已经失去了锁的保护，不再是线程安全的了。于是，后面的 xxx, yyy, zzz 会面临竞争的危险。另外一个问题是，由于锁已经被释放，此时，调用者写的那行 unlock 可能导致未知的行为，这取决于 Mutex 的设计。
解决的方法其实也很简单，库函数 waitCond 既然释放了锁，那也就得需要它再锁上：



所以，我们需要将 unlock 与 waitCond 原子化. 我们先将释放锁，进入等待这两个操作封装到一个函数里面，然后在这个函数内部去原子化，也只有这样才可能实现原子化。封装之后的 waitCond 可能是这样的：



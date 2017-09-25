#条件变量的虚假唤醒
#概述
条件变量存在虚假唤醒问题(spurious wakeups)，即当一个线程唤醒时，可能并不是条件变量保护的条件得到了满足。[WikiPedia](https://en.wikipedia.org/wiki/Spurious_wakeup) 中的描述如下：
> Spurious wakeup describes a complication in the use of condition variables as provided by certain multithreading APIs such as POSIX Threads and the Windows API.
Even after a condition variable appears to have been signaled from a waiting thread's point of view, the condition that was awaited may still be false. One of the reasons for this is a spurious wakeup; that is, a thread might be awoken from its waiting state even though no thread signaled the condition variable. 

spurous wakeup 描述了一种在 POSIX 或 Windows 线程库中使用线程变量时可能存在的混乱行为。
即使在等待线程看来，其等待的条件变量已经被唤醒，然而与条件变量相关联的被保护条件仍然可能是 false。
所以，在没有其它线程 signal 之前，一个线程仍可能从等待状态被唤醒。

#原理
为什么会发生虚假唤醒呢？甚至有一些人质疑，是否在实际中不会发生虚假唤醒。
Joe Duffy's Concurrent Programming On Windows 一书中有这一段描述：
> Note that in all of the above examples, threads must be resilient to something called spurious wake-ups - code that uses condition variables should remain correct and lively even in cases where it is awoken prematurely, that is, before the condition being sought has been established. This is not because the implementation will actually do such things (although some implementations on other platforms like Java and Pthreads are known to do so), nor because code will wake threads intentionally when it's unnecessary, but rather due to the fact that there is no guarantee around when a thread that has been awakened will become scheduled. Condition variables are not fair. It's possible - and even likely - that another thread will acquire the associated lock and make the condition false again before the awakened thread has a chance to reacquire the lock and return to the critical region.

最关键的一段是：
> This is not because the implementation will actually do such things (although some implementations on other platforms like Java and Pthreads are known to do so), nor because code will wake threads intentionally when it's unnecessary, but rather due to the fact that there is no guarantee around when a thread that has been awakened will become scheduled.

即，导致虚假唤醒的真正原因是，被唤醒的线程并不是马上被调度的。有可能这个已经处于唤醒状态的线程还排列在线程调度队列中等待被调度。进一步地，另外一个线程可能抢到了 condWait 中释放的锁进入处理并可能更改条件，于是当前一个线程再次被调度时，条件可能不再满足了。
从代码的角度来说明：

```
//waiter thread
lock(mutex);    //1

////////////// three atomic parts of condWait /////////////////
unlock(mutex);
switchToCoreAndWait(cond);
lock(mutex);
//////////////////////////////////////////////////////////////


unlock(mutex);    //3
```

#深入

#解决


[1]: https://stackoverflow.com/questions/1461913/does-c-sharp-monitor-wait-suffer-from-spurious-wakeups/1461956#1461956 
[2]: https://groups.google.com/forum/?hl=de#!topic/comp.programming.threads/MnlYxCfql4w
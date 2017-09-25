#条件变量的虚假唤醒
##概述
条件变量存在虚假唤醒问题(spurious wakeups)，即当一个线程唤醒时，可能并不是条件变量保护的条件得到了满足。[WikiPedia](https://en.wikipedia.org/wiki/Spurious_wakeup) 中的描述如下：
> Spurious wakeup describes a complication in the use of condition variables as provided by certain multithreading APIs such as POSIX Threads and the Windows API.
Even after a condition variable appears to have been signaled from a waiting thread's point of view, the condition that was awaited may still be false. One of the reasons for this is a spurious wakeup; that is, a thread might be awoken from its waiting state even though no thread signaled the condition variable. 

spurous wakeup 描述了一种在 POSIX 或 Windows 线程库中使用线程变量时可能存在的混乱行为。
即使在等待线程看来，其等待的条件变量已经被唤醒，然而与条件变量相关联的被保护条件仍然可能是 false。
所以，在没有其它线程 signal 之前，一个线程仍可能从等待状态被唤醒。

##原理
为什么会发生虚假唤醒呢？甚至有一些人质疑，是否在实际中不会发生虚假唤醒。
Joe Duffy's Concurrent Programming On Windows 一书中有这一段描述：
> Note that in all of the above examples, threads must be resilient to something called spurious wake-ups - code that uses condition variables should remain correct and lively even in cases where it is awoken prematurely, that is, before the condition being sought has been established. This is not because the implementation will actually do such things (although some implementations on other platforms like Java and Pthreads are known to do so), nor because code will wake threads intentionally when it's unnecessary, but rather due to the fact that there is no guarantee around when a thread that has been awakened will become scheduled. Condition variables are not fair. It's possible - and even likely - that another thread will acquire the associated lock and make the condition false again before the awakened thread has a chance to reacquire the lock and return to the critical region.

最关键的一段是：
> This is not because the implementation will actually do such things (although some implementations on other platforms like Java and Pthreads are known to do so), nor because code will wake threads intentionally when it's unnecessary, but rather due to the fact that there is no guarantee around when a thread that has been awakened will become scheduled.

即，导致虚假唤醒的真正原因是，被唤醒的线程并不是马上被调度的。有可能这个已经处于唤醒状态的线程还排列在线程调度队列中等待被调度。进一步地，另外一个线程可能抢到了 condWait 中释放的锁进入处理并可能更改条件，于是当前一个线程再次被调度时，条件可能不再满足了。
从代码的角度来说明：

```
/**********waiter thread**********/
lock(mutex);    //1
if(!condIsTrue)
{
    //waitCond(condVar, mutex);
    ////////////// three atomic parts of condWait /////////////////
    unlock(mutex);    //2.1
    switchToCoreAndWait(condVar);    //2.2
    lock(mutex);    //2.3
    //////////////////////////////////////////////////////////////
}

assert(condIsTrue);    //3
//do something while condIsTrue
condIsTrue=false;     //4，条件为真，做一些事情后重置为 flase，保证只处理一次
unlock(mutex);        //5
```


```
/**********signaler thread**********/
lock(mutex);    
condIsTrue = true;
signalCond(condVar);
unlock(mutex); 
```
如上，设线程 A 在第 2.2 行代码处阻塞以等待 condIsTrue 为真。设线程 B 是唤醒线程，它做以下事情：
- 更改了条件，设置为 true
- 唤醒线程
- 释放用户锁


做唤醒线程后，线程 A 被重新加入到线程调度队列中(此时线程 A 不会立即运行)。若此时正好有另一个等待线程 C，执行第 1 行代码，顺利拿到锁并且往后面执行直到它重置 condIsTrue 为 false(注意线程 C 因为 condIsTrue 为 true 故不会执行 waitCond)。当线程 A 被调度，获得了锁并往后执行时，condIsTrue 的值已经为 flase 了，assert(condIsTrue) 这行会报错。这个问题被称为“虚假唤醒”。

##解决
上一篇文章[“条件变量的设计与实现”](https://juniorfans.gitbooks.io/blog/content/2-condvar.html)时，指出一个问题：等待线程释放锁并进入等待原子化，与此处虚假唤醒问题有异曲同工之妙。

- **释放锁并进入等待原子化**：unlock 和 switchToCoreAndWait 之间可能被其它线程抢占执行，因 unlock 后锁已释放

- **虚假唤醒**：switchToCoreAndWait 和 lock 之间可能被其它线程抢占执行，因 switchToCoreAndWait 后锁已释放

有了解决前一个问题的经验，我们大致可以猜测，后一个问题也不那么简单，至少，如果我们通过加入另外一个锁的手段，可能引发死锁问题，这就像上面提到的那篇文章描述的“**将钥匙投进了上了锁的个人信箱里面**”。
review 这个问题，发现关键点在于：**当有一个线程即将被唤醒即将获得锁时，禁止其它线程获得同一个锁**。像上一个问题的解决方案：利用已存在的 happen-before 关系标记这种场景，在后续发生的线程中识别并处理这种异常。
然而，因为对用户锁加锁或解锁已经脱离了条件变量的范畴：`pthread_mutex_lock` 与 `pthread_mutex_unlock` 只接受一个 mutex，所以我们可能需要更改 mutex 的设计以解决这个问题。当然可能存在其它的解决方案，但我预期应该更复杂。
David R. Butenhof in "Programming with POSIX Threads" (p. 80):
> Spurious wakeups may sound strange, but on some multiprocessor systems, making condition wakeup completely predictable might substantially slow all condition variable operations.

虚假唤醒导致的结果就是，被唤的线程在调度时发现条件并没有满足。因此，我们只需要在调用代码中处理这个例外就可以了：

```
/**********waiter thread**********/
lock(mutex);    //1
while(!condIsTrue)    //将 if 改为 while
{
    //waitCond(condVar, mutex);
    ////////////// three atomic parts of condWait /////////////////
    unlock(mutex);    //2.1
    switchToCoreAndWait(condVar);    //2.2
    lock(mutex);    //2.3
    //////////////////////////////////////////////////////////////
}

assert(condIsTrue);    //3
//do something while condIsTrue
condIsTrue=false;     //4，条件为真，做一些事情后重置为 flase，保证只处理一次
unlock(mutex);        //5
```


```
/**********signaler thread**********/
lock(mutex);    
condIsTrue = true;
signalCond(condVar);
unlock(mutex); 
```

实际上，这也是标准的做法，不论是在 glibc 上，还是在 Windows API 上。

##一些有用的参考
[does-c-sharp-monitor-wait-suffer-from-spurious-wakeups][1]
[comp.programming.threads][2]

[1]: https://stackoverflow.com/questions/1461913/does-c-sharp-monitor-wait-suffer-from-spurious-wakeups/1461956#1461956 
[2]: https://groups.google.com/forum/?hl=de#!topic/comp.programming.threads/MnlYxCfql4w
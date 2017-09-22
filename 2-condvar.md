1.wait 里面先增加了 \_\_wseq\(waiter sequence\)和 waiter reference，达到注册“等待者”的目的，再释放锁。于是在 signal 中可以观察到等待者。

2.每个 waiter 增加 \_\_wseq 后得到的值即是这个 waiter 在等待队列中的位置。

3.wait 中使用的 futex\_wait 关键字与 signal 中 futex\_wake 是一样的关键字: \_\_g\_signals。\_\_g\_signals 大于0表示可唤醒等待者。futex\_wait 期望该关键字值为0，futex\_wake 设置该关键字值为1.

4.使用两组机制：G1 组由那些有机会消耗 signals 的 waiters 组成。新到来的 signals 将一直唤醒该组的 waiters 直到 G1 所有的 waiters 都被唤醒。G2 组由后到达的 waiters 组成\(此时 G1 中还存在未被唤醒的 waiters\)。当 G1 中所有的 waiters 都被唤醒且有一个新的 signal 到达，则这个 signal 将 G2 转化为新的 G1

5.wait 中释放锁后，自旋等待，检查 \_\_g\_signals，自旋次数结束，进入 futex\_wait。（省掉掉了异常处理：惊群效应，取消等待，组关闭等）

6.signal 中获得锁后，检查 \_\_wseq，当没有等待者直接返回。否则获得锁，检查是否需要切换组\(例如首次调用 wait 后 G1 为空，G2有一个等待者，则首次调用 signal 后需要将 G2 切换为 G1\)，递增 \_\_g\_signals，递减 \_\_g\_size\(未唤醒的 waiters 个数\)，再调用 futex\_wake。

7.由5和6可知，若线程A wait，线程B signal，有以下破坏“释放锁并等待”的执行顺序：“A-释放锁，B-获得锁，B-递增 \_\_g\_signals，B-futex\_wake，A-futex\_wait”，该执行顺序下，最后一个 A-futex\_wait由于 futex\_wait 期望的关键字 \_\_g\_signals 值不为 0 则它不会进入等待，被直接唤醒；若执行的顺序是，“A-释放锁，B-获得锁，B-递增 \_\_g\_signals，A-futex\_wait，B-futex\_wake”，由原因同上，futex\_wait 并不会真正进入等待。这两种情况下的 futex\_wake 没有任何作用\(它本来不会引起阻塞，调用无害\)！


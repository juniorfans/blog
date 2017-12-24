#MergingIterator
这个迭代器可以简单看成一个混合的迭代器，它对外提供 Prev, Next, Seek 接口,
对内的实现是管理着一堆 IteratorWrapper, 通过包装它们的 Prev, Next, Seek 接口实现功能。
IteratorWrapper 前面已经介绍过了，它是一个 Iterator 子类的包装。
***MergingIterator*** 有如下设计：
- 1.当迭代器有相同的 InternalKey 时它们会被多次遍历到。
- 2.若 MergingIterator 中两个迭代器 x, y 的当前位置键值一样，则谁在 MergingIterator 中下标小则更小。否则更大。
- 3.“反方向遍历”的设计是，保证此次遍历之后的键值不等于当前键值。
- 4."Next 之后 Prev" 遍历的设计是，定位到比 MergingIterator当前键值小的最大那个位置。
- 5."Prev 之后 Next" 遍历的设计是，定位到比 MergingIterator当前键值大的最小那个位置。
- 6.由3，4两点可知，调用Next再调用Prev 不一定能回到之前的那个位置

##1.1 SeekToFirst
这个接口对外提供的功能是，定位到最小的位置，需要将维持的 n\_ 个迭代器 WrapperIterator 全部 seek 到 first 位置，
并且通过比较这 n\_ 个迭代器 first 位置的值, 找到最小键值的迭代器, 让 current\_ 指向它。
##1.2 SeekToLast
逻辑同 SeekToFirst，先将 n\_ 个迭代器 seek 到 last 位置，找到最大键值的迭代器，让 current\_ 指向它。
##1.3 Seek(const Slice& target)
定位到首个键值大于 target 的位置。先将 n\_ 个迭代器 seek 到首次各自键值大于等于 target 的位置，再比较它们此时各自键值，
找到最小键值的迭代器，让 current\_ 指向它。
注意 leveldb 整个体系中，查找一个键值的逻辑均是：寻找到首个键值大于等于目标键的位置。本文 2.3 节中有指出这点。
##1.4 Next
这个接口的实现有些复杂。
1.如果上一次是向后遍历的（即与 Next 方向一致），现在调用 Next，我们需要计算 current\_ 所在迭代器的下一个位置的键值与其它
迭代器位置的键值中最小的那个，即是整个迭代器组的 Next 位置。
2.如果上一次是向前遍历的（与 Next 方向相反），现在调用 Next，则将所有除 current\_ 的分迭代器都 seek 到 current\_.key() 的位置。
再判断若某些分迭代器位置有效且的键值为 current\_.key() (即不大于 current\_.key())则对分迭代器调用 Next。再使用 FindSmallest 找出最小的那个分迭代器，让 current\_ 指向它。

##1.5 Prev
与 Next 相对应，
1.如果上一次是向前遍历的（即与 Prev 方向一致），现在调用 Prev，我们需要计算 current\_ 所在迭代器的上一个位置的键值与其它
迭代器位置的键值中最大的那个，即是整个迭代器组的 Prev 位置。
2.如果上一次是向后遍历的（与 Prev 方向相反），现在调用 Prev，则将所有除 current\_ 的迭代器都 seek 到 current\_.key() 的位置。
若分迭代器位置有效则调用对分迭代器 Prev。再使用 FindLargest 找出最大的那个迭代器，让 current\_ 指向它。

注意 Prev 与 Next 不同的地方在于，当它们都处于反方向遍历时，且都 seek 到了 current\_.key() 的位置，Next 中还需要判断各迭代器当前位置键是否等于 current\_.key()，这是为了保证 MergingIterator 设计的第3条。所有迭代器的 seek(target) 的逻辑是，找到键值大于等于 target 的位置。若已经大于 current\_.key() 了则不需要对分迭代顺再调用 next 以满足第3条。

##1.6 图示
如下图x, y, z 表示 MergingIterator 的三个分迭代器，这三个迭代器各三个键(InternalKey)，三个分迭代器的键对应相等：它们的 user\_key 都一样，版本号分别为19，18，17，注意 InternalKey 的排序规则，版本号较大的排在前面(Prev 方向)。x18 表示 x 那个分迭代器的当前位置是18那个位置。红色三角符号表示 current\_ 指向的那个迭代器的当前位置。红色的线条把各个分迭代器的当前位置串联起来了。图演示了迭代器 Next 和 Prev 被调用时的状态变化。
依次调用 Seek(18), Next, Prev, Prev, Next 迭代器的状态变化。
![文本状态变化](/assets/leveldb/mergerIterator_1.JPG)

![图示状态变化](/assets/leveldb/mergerIterator_2.JPG)


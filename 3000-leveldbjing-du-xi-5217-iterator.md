#leveldb 的 Iterator
##概述
本文主要讲述 leveldb 的 Iterator, 包括 Block::Iter， DBIter， MergingIterator， TwoLevelIterator， LevelFileNumIterator，IteratorWrapper。
给出一些场景及 Iterator：
- leveldb 是数据库，我们需要提供查询能力及遍历能力，这需要 DBIter
- 将内存中的 Table 写入数据库时要遍历 Table，这需要 TwoLevelIterator
- 


预计包含的内容：如何遍历一个 Block，如何遍历一个 Table，考虑在有缓存，快照的情况下，遍历整个数据库。
##Iterator
leveldb::Iterator 是一个抽象类，和一般的迭代器没有太大的不同，同样是提供了以下接口

```
virtual bool Valid() const
virtual void Seek(const Slice& target)
virtual void SeekToFirst() 
virtual void SeekToLast()
virtual void Next() 
virtual void Prev() 
Slice key() const 
Slice value() const
virtual Status status()
```
*seek* 系列函数很明显：定位，与此类似的还有 **Next**，**Prev**。**key** 和 **value** 更明显，返回当前节点的键值对。 **Valid** 和 **status** 函数是迭代器的状态(有效｜无效)。

##Block::Iter
这个迭代器用来遍历 Block，Block 的格式见 [3003-leveldb精读系列-format][1]，Block 中的 entry 是按 key 的顺序来排列的，这一点需要时时在心中铭记。
Block::Iter 迭代器的实现并不是基于在内存中构建 Block 已经解析好的 entry 链表或者数组，而是直接在字节流上建立起来的。维持着当前 Block 的字节流，当前 entry 字节流的偏移位置，重启点(每组起始位置)数组存储的起始位置，重启点个数，当前 entry 所在的组的下标。
Block::Iter 实现如下：
Next：
- 1.通过当前 entry 的最后一个元素 value 的位置及长度，计算并移动到下一个 entry 的起始地址
- 2.调用 DecodeEntry 解析当前 entry 数据
- 3.重置迭代器中的位置， key, value, 当前 entry 所在组的下标 restart_index

Prev：
- 1.计算前面一个 entry 所在的那个组，原代码中使用了循环，对比每个组的首个 entry 地址与当前 entry 位置比较，找到首个地址小于 entry 位置的最大的组，即是前一个 entry 所在的组。但个人感觉不是必须的，这一点如果有人发现了原因烦请告之(1127365587@qq.com)。
- 2.使用 SeekToRestartPoint 将迭代器定位到第1步中找到的那个组的首地址
- 3.在这个组中循环向后找，解析每个 entry 并设置迭代器位置， key, value 等信息，直至遇到当前 entry 即停止。此时最后一个被解析即是前一个 entry。详见代码中的 do while

Seek(const Sliece& target)：
- 1.使用二分法找到 target 所在的组: 即小于 target 的最大组
- 2.使用 SeekToRestartPoint 将迭代器定位到第1步中找到的那个组的首地址
- 3.从第2步中迭代器的位置循环向后遍历，解析，直到遇到一个 entry 的键值 >= target



那么遍历 Block 






[1]: 3003-leveldb精读系列-format
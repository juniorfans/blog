#leveldb 的 Iterator
##概述
本文主要讲述 leveldb 的 Iterator, 包括 Block::Iter， MergingIterator， TwoLevelIterator， LevelFileNumIterator，IteratorWrapper。
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
*seek* 系列函数很明显：定位，与此类似的还有 **Next**，**Prev**。**key** 和 **value** 更明显，返回当前节点的键值对。 

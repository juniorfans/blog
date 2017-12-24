#leveldb 的 Iterator
##概述
leveldb 的 Iterator, 涉及 Block::Iter，LevelFileNumIterator，IteratorWrapper，MergingIterator，TwoLevelIterator，DBIter。我们以一组文章的形式依次讲解。

先给出一些场景及 Iterator：
- leveldb 是数据库，我们需要提供查询能力及遍历能力，这需要 DBIter。
- Block 是 leveldb 中低层次数据源，考虑到磁盘存储和索引，我们需要 Block::Iter 遍历。
- 将内存中的 Table 写入数据库时要遍历 Table，Table 是一个较高层次的数据结构，处于较低层次提供数据服务的是 Block。TAble 包含了键到Block的映射索引，而 Block 内部也有索引：这是一个多级索引结构，我们需要 TwoLevelIterator 遍历。
- 整个 leveldb 迭代器体系，均是以 Iterator 为基类，实现它们的虚函数。为了提高效率，使用IteratorWrapper  去减少虚函数调用次数。
- 一个 Table 有多个 Block，为了遍历的方便，我们需要一个抽象的迭代器，而不是管理一堆具体的迭代器，因此使用 MergingIterator 去抽象迭代器。
等等。

预计包含的内容：如何遍历一个 Block，如何遍历一个 Table，考虑在有缓存，快照的情况下，遍历整个数据库。

##接口
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







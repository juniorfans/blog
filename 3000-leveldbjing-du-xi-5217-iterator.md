#leveldb 的 Iterator
##概述
本文主要讲述 leveldb 的 Iterator, 包括 Block::Iter，LevelFileNumIterator，IteratorWrapper，MergingIterator，TwoLevelIterator，DBIter。
给出一些场景及 Iterator：
- leveldb 是数据库，我们需要提供查询能力及遍历能力，这需要 DBIter
- 将内存中的 Table 写入数据库时要遍历 Table，这需要 TwoLevelIterator
- 


预计包含的内容：如何遍历一个 Block，如何遍历一个 Table，考虑在有缓存，快照的情况下，遍历整个数据库。
##1 Iterator
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

##2 Block::Iter
这个迭代器用来遍历 Block，Block 的格式见 [3003-leveldb精读系列-format][1]，Block 中的 entry 是按 key 的顺序来排列的，这一点需要时时在心中铭记。
Block::Iter 迭代器的实现并不是基于在内存中构建 Block 已经解析好的 entry 链表或者数组，而是直接在字节流上建立起来的。维持着当前 Block 的字节流，当前 entry 字节流的偏移位置，重启点(每组起始位置)数组存储的起始位置，重启点个数，当前 entry 所在的组的下标。
Block::Iter 实现如下：
###2.1 Next：
- 1.通过当前 entry 的最后一个元素 value 的位置及长度，计算并移动到下一个 entry 的起始地址
- 2.调用 DecodeEntry 解析当前 entry 数据
- 3.重置迭代器中的位置， key, value, 当前 entry 所在组的下标 restart_index

###2.2 Prev：
- 1.计算前面一个 entry 所在的那个组，原代码中使用了循环，对比每个组的首个 entry 地址与当前 entry 位置比较，找到首个地址小于 entry 位置的最大的组，即是前一个 entry 所在的组。但个人感觉不是必须的，这一点如果有人发现了原因烦请告之(1127365587@qq.com)。
- 2.使用 SeekToRestartPoint 将迭代器定位到第1步中找到的那个组的首地址
- 3.在这个组中循环向后找，解析每个 entry 并设置迭代器位置， key, value 等信息，直至遇到当前 entry 即停止。此时最后一个被解析即是前一个 entry。详见代码中的 do while

###2.3 Seek(const Sliece& target)：
- 1.使用二分法找到 target 所在的组: 即小于 target 的最大组
- 2.使用 SeekToRestartPoint 将迭代器定位到第1步中找到的那个组的首地址
- 3.从第2步中迭代器的位置循环向后遍历，解析，直到遇到一个 entry 的键值 >= target。***这里判断不仅仅等于，而是大于等于***的原因需要参考 [3004-leveldb精读系列-key][2]，因为 key 的排序规则是 user\_key正序 --> sequenceNumber降序 --> valueType降序，而传入 Seek 函数的 key 会组装一个最新的版本号(大于一切 sstable, memtable, cache 中的版本号)，所以找到的同 user_key 的键一定排在它后面。

***SeekToFirst*** 和 ***SeekToLast*** 比较简单，就不再一一阐述。可以看下面的代码注释

```
class Block::Iter : public Iterator {
 private:
  const Comparator* const comparator_;

  //lzh: 所在的 block 
  const char* const data_;      // underlying block contents

  //lzh: restart array 存储的是 restart number 数组, 固长编码. restarts_ 即是这个数组起始地址相对 data_ 的偏移
  uint32_t const restarts_;     // Offset of restart array (list of fixed32)

  //lzh: 总共有多少个 restart number
  uint32_t const num_restarts_; // Number of uint32_t entries in restart array

  // current_ is offset in data_ of current entry.  >= restarts_ if !Valid
  //lzh: 当前 entry 在它的块中(data_)的偏移量
  uint32_t current_;

  //lzh: current_ entry 所在组的下标，即当前 entry 的 restart 索引号
  uint32_t restart_index_;  // Index of restart block in which current_ falls

  std::string key_;
  Slice value_;
  Status status_;

  inline int Compare(const Slice& a, const Slice& b) const {
    return comparator_->Compare(a, b);
  }

  // Return the offset in data_ just past the end of the current entry.
  //lzh: 由当前 entry 的最后一个元素 value 及长度计算下一个 entry 的开始位置
  //lzh: 下一个 entry 的偏移. entry 的最后一部分是 value_data, 所以计算下一个 entry 的偏移可以按下面的方法计算
  inline uint32_t NextEntryOffset() const {
    return (value_.data() + value_.size()) - data_;
  }

  /************************************************************************/
  /* 
	lzh: 获得第 index 个组的偏移，这个值即是 restarts 数组的第 index 个元素的值
  */
  /************************************************************************/
  uint32_t GetRestartPoint(uint32_t index) {
    assert(index < num_restarts_);
    return DecodeFixed32(data_ + restarts_ + index * sizeof(uint32_t));//restarts_ 是 restarts 数组起始地址相对 data_ 的偏移
  }

  //lzh: 指向索引为 index 的 restart 位置. 注意此时 key_ 和 value_ 都要清空
  void SeekToRestartPoint(uint32_t index) {
    key_.clear();
    restart_index_ = index;
    // current_ will be fixed by ParseNextKey();

    // ParseNextKey() starts at the end of value_, so set value_ accordingly
    uint32_t offset = GetRestartPoint(index);
    value_ = Slice(data_ + offset, 0);
  }

 public:
  Iter(const Comparator* comparator,
       const char* data,
       uint32_t restarts,
       uint32_t num_restarts)
      : comparator_(comparator),
        data_(data),
        restarts_(restarts),
        num_restarts_(num_restarts),
        current_(restarts_),
        restart_index_(num_restarts_) {
    assert(num_restarts_ > 0);
  }

  virtual bool Valid() const { return current_ < restarts_; }
  virtual Status status() const { return status_; }
  virtual Slice key() const {
    assert(Valid());
    return key_;
  }
  virtual Slice value() const {
    assert(Valid());
    return value_;
  }

  virtual void Next() {
    assert(Valid());
    ParseNextKey();
  }

  /************************************************************************/
  /* 
	lzh: 若当前已经是第一个 entry 了，再调用 Prev 则会到失效的位置
  */
  /************************************************************************/
  virtual void Prev() {
    assert(Valid());

    // Scan backwards to a restart point before current_
    const uint32_t original = current_;
	
	//lzh: [疑问] 这里为什么要用循环呢？是不是将 while 换成 if 即可?
	//lzh: 获得小于 original 的最大的重启点/组位置，换种说法就是找到 original 所在的那个组
    while (GetRestartPoint(restart_index_) >= original) {
      if (restart_index_ == 0) {
		  //lzh: GetRestartPoint(0) >= original，表示第 0 个重启点的位置大于当前 entry 位置
		  //lzh: 即说明当前是第0个 entry，此时也必有 GetRestartPoint(0) == original
		  //lzh: 说明当前迭代器处于第0个位置, 再往前则跳到最未的失效位置
        // No more entries

		//lzh: current_ 指向 restarts 数组位置, 即第一个非 entry 字节的位置, 此时 Valid() 函数将返回 false
        current_ = restarts_;
		//lzh: 首个失效的 restart 数组下标（有效范围是 0 ~ num_restarts_-1）之后只要调用 GetRestartPoint 则会 assert 失败
        restart_index_ = num_restarts_;
        return;
      }
      restart_index_--;
    }

	//lzh: 找到了 original 所使用的那个重启点: 第 restart_index_ 个重启点.
	//lzh: 从第 restart_index_ 个重启点指示的那个 entry 开始往后遍历直到遇到 original，此时刚好将 original 整个 key 复原出来
    SeekToRestartPoint(restart_index_);
    do {
      // Loop until end of current entry hits the start of original entry
    } while (ParseNextKey() && NextEntryOffset() < original);
  }

  /************************************************************************/
  /* 
	lzh: 定位到最后一个比 target 小的 entry，(即小于 target 的最大的 entry)
  */
  /************************************************************************/
  virtual void Seek(const Slice& target) {
    // Binary search in restart array to find the first restart point
    // with a key >= target
	  //lzh: 使用二分法找到那个仅小于 target 的重启点
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      const char* key_ptr = DecodeEntry(data_ + region_offset,
                                        data_ + restarts_,
                                        &shared, &non_shared, &value_length);
      if (key_ptr == NULL || (shared != 0)) {
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }

    // Linear search (within restart block) for first key >= target
    SeekToRestartPoint(left);
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }

  virtual void SeekToFirst() {
	  //lzh: 直接定位到第一个组的位置，即是该 Block 的首个 entry 位置
    SeekToRestartPoint(0);
    ParseNextKey();
  }

  virtual void SeekToLast() {
    //lzh: 定位到最后一个组的位置
	SeekToRestartPoint(num_restarts_ - 1);
	
	//lzh: 遍历这个组找到最后一个 entry。
	//lzh: 注意最后一个组的最后一个 entry 也即整个 Block 的最后一个 entry 它的后面即是 restarts，这里判断不要过界了
    while (ParseNextKey() && NextEntryOffset() < restarts_) {
      // Keep skipping
    }
  }

 private:
  void CorruptionError() {
    current_ = restarts_;
    restart_index_ = num_restarts_;
    status_ = Status::Corruption("bad entry in block");
    key_.clear();
    value_.clear();
  }

  /************************************************************************/
  /* 
	lzh: 
	1.通过当前 entry 的 value_data 位置和 value_data 的长度可以算出下一个 entry的起始位置。
	2.解析下一个 entry 的字节流，将其结构化，算出 entry 中的各个字段之值。
	3.最后再计算出这个 entry 所在的组的下标
  */
  /************************************************************************/
  bool ParseNextKey() {
    current_ = NextEntryOffset();
    const char* p = data_ + current_;

	//lzh: 所有 entry 排布完, 后面排的是 restarts_ 数组.
    const char* limit = data_ + restarts_;  // Restarts come right after data
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }

    // Decode next entry
    uint32_t shared, non_shared, value_length;
	//lzh: 解析当前 entry, 解析完之后，p 指向当前 entry 的 unshared_key_data
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
	//lzh: 如上, 这不可能为 NULL，shared 是当前 entry 的键与前一个 entry 的键 key_ 的相同前缀的长度, 所以必然小于 key_ 长度
    if (p == NULL || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
		//lzh: key_ 是上一个 entry 的键，此处相当于让 key_ 只保留与后一个 entry 共同的前缀
      key_.resize(shared);
	  //lzh: 再加上当前 entry 键中不共同的部分, key_ 即变成了当前 entry 的键
      key_.append(p, non_shared);
	  //lzh: p+non_shared 即指向了当前 entry 的 value_data
      value_ = Slice(p + non_shared, value_length);

	  //lzh: restart_index_ 目前指向的是上一个 entry 所在的组的下标，现在需要调整。
	  //lzh: 考虑到本函数的调用场景有可能跨越多个 entry: 如 SeekToFirst, SeekToLast
	  //lzh: 所以新的位置可能需要跨越多个组，因此下面使用了一个循环去跨越组，直到定位到当前 entry 所在的组，也即 restart_index
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {
        ++restart_index_;
      }
      return true;
    }
  }
};
```
##3 LevelFileNumIterator
这个迭代器用于遍历某层的 sst 文件。迭代器维持了一个指向文件的下标 index_，通过它的增减表示向前后的遍历。
index_ 等于文件总个数时表示迭代器失效。
下面是迭代器主要方法的实现
###3.1 Next
判断当前迭代器有效性。直接增加 index_
###3.2 Prev
判断当前迭代器有效性。若当前 index_ 为0则设置为文件个数，表示失效。直接减小 index_
###3.3 Seek(const Slice& target)
使用 FindFile，通过二分法从文件列表中找出首个 largest >= target 的文件，即说明键 target 在这个文件之中。
设置 index_ 为此文件下标即是。
详情见代码注释

```
class Version::LevelFileNumIterator : public Iterator {
 public:
  LevelFileNumIterator(const InternalKeyComparator& icmp,
                       const std::vector<FileMetaData*>* flist)
      : icmp_(icmp),
        flist_(flist),
        index_(flist->size()) {        // Marks as invalid
  }
  virtual bool Valid() const {
    return index_ < flist_->size();
  }

  /************************************************************************/
  /* 
	lzh:
	返回首个 largest 比 target 的文件.
	若 target < flist_[0].smallest 则返回 0，该情况下表示 target 可能在 flist_[0] 中或者不在
	若 target > flist_[flist_->size()-1] 则返回 flist_->size()，无效的位置
  */
  /************************************************************************/
  virtual void Seek(const Slice& target) {
    index_ = FindFile(icmp_, *flist_, target);
  }
  virtual void SeekToFirst() { index_ = 0; }
  virtual void SeekToLast() {
    index_ = flist_->empty() ? 0 : flist_->size() - 1;
  }
  virtual void Next() {
    assert(Valid());
    index_++;
  }
  virtual void Prev() {
    assert(Valid());
    if (index_ == 0) {
      index_ = flist_->size();  // Marks as invalid
    } else {
      index_--;
    }
  }

  /************************************************************************/
  /* 
	lzh: 返回当前位置文件的 largest
  */
  /************************************************************************/
  Slice key() const {
    assert(Valid());
    return (*flist_)[index_]->largest.Encode();
  }

  /************************************************************************/
  /* 
	lzh: 返回当前位置文件的 number, size 的固长编码
  */
  /************************************************************************/
  Slice value() const {
    assert(Valid());
    EncodeFixed64(value_buf_, (*flist_)[index_]->number);
    EncodeFixed64(value_buf_+8, (*flist_)[index_]->file_size);
    return Slice(value_buf_, sizeof(value_buf_));
  }
  virtual Status status() const { return Status::OK(); }
 private:
  const InternalKeyComparator icmp_;
  const std::vector<FileMetaData*>* const flist_;
  uint32_t index_;

  // Backing store for value().  Holds the file number and size.
  mutable char value_buf_[16];
};
```
***FindFile*** 是一个二分法查找算法，如下：

```
/************************************************************************/
/************************************************************************/
/* 
	lzh: 使用二分法找出 files(正序排列) 中首个 largest >= key 的 file
	注意边界条件
		1.若 files[0].smallest > key 则返回 0
		2.若 files[files.size()-1].largest < key 则返回 files.size()
	调用者若想调用此函数返回 key 所在的文件，则需要注意第 1 种情况可能是个反例
*/
/************************************************************************/
int FindFile(const InternalKeyComparator& icmp,
             const std::vector<FileMetaData*>& files,
             const Slice& key) {
  uint32_t left = 0;
  uint32_t right = files.size();
  while (left < right) {
    uint32_t mid = (left + right) / 2;
    const FileMetaData* f = files[mid];
    if (icmp.InternalKeyComparator::Compare(f->largest.Encode(), key) < 0) {
      // Key at "mid.largest" is < "target".  Therefore all
      // files at or before "mid" are uninteresting.
      left = mid + 1;
    } else {
      // Key at "mid.largest" is >= "target".  Therefore all files
      // after "mid" are uninteresting.
      right = mid;
    }
  }
  return right;
}
```
##4 IteratorWrapper
这是一个迭代器包装器，它包装了 Iterator 接口, 存储着 Iterator.key 和 Iterator.value。这个类主要为效率计：leveldb 中所有的迭代器都继承了 Iterator 类，实现了它里面的虚函数，我们在代码中广泛地使用了虚函数动态绑定特性，这当然很方便，但是虚函数调用的效率比普通函数调用会低一些，因为虚函数地址需要在运行期决议出来：通过访问对象前四个字节所指向的虚表，取出目标函数地址再执行 call 指令。而普通函数调用的函数地址在编译完成时就已经确定下来了，显然效率更高。
我们将 key 和 valid 存储下来，减少虚函数的调用次数。实现如下：

```
// A internal wrapper class with an interface similar to Iterator that
// caches the valid() and key() results for an underlying iterator.
// This can help avoid virtual function calls and also gives better
// cache locality.
// lzh: IteratorWrapper 包装了 Iterator 接口, 存储着 Iterator.key() 和 Iterator.valid()
// lzh: 这个类主要为效率计：避免多次调用虚函数的损耗，将 key 和 valid 存储下来
class IteratorWrapper {
 public:
  IteratorWrapper(): iter_(NULL), valid_(false) { }
  explicit IteratorWrapper(Iterator* iter): iter_(NULL) {
    Set(iter);
  }
  ~IteratorWrapper() { delete iter_; }
  Iterator* iter() const { return iter_; }

  // Takes ownership of "iter" and will delete it when destroyed, or
  // when Set() is invoked again.
  void Set(Iterator* iter) {
    delete iter_;
    iter_ = iter;
    if (iter_ == NULL) {
      valid_ = false;
    } else {
      Update();
    }
  }


  // Iterator interface methods
  bool Valid() const        { return valid_; }
  Slice key() const         { assert(Valid()); return key_; }
  Slice value() const       { assert(Valid()); return iter_->value(); }
  // Methods below require iter() != NULL
  Status status() const     { assert(iter_); return iter_->status(); }
  void Next()               { assert(iter_); iter_->Next();        Update(); }
  void Prev()               { assert(iter_); iter_->Prev();        Update(); }
  void Seek(const Slice& k) { assert(iter_); iter_->Seek(k);       Update(); }
  void SeekToFirst()        { assert(iter_); iter_->SeekToFirst(); Update(); }
  void SeekToLast()         { assert(iter_); iter_->SeekToLast();  Update(); }

 private:
  void Update() {
    valid_ = iter_->Valid();
    if (valid_) {
      key_ = iter_->key();
    }
  }

  Iterator* iter_;
  bool valid_;
  Slice key_;
};
```

##5 MergingIterator
这个迭代器可以简单看成一个混合的迭代器，它对外提供 Prev, Next, Seek 接口, 
对内的实现是管理着一堆 IteratorWrapper, 通过包装它们的 Prev, Next, Seek 接口实现功能。
IteratorWrapper 上面已经介绍过了，它是一个 Iterator 子类的包装。
###5.1 SeekToFirst
这个接口对外提供的功能是，定位到最小的位置，需要将维持的 n\_ 个迭代器 WrapperIterator 全部 seek 到 first 位置，
并且通过比较这 n\_ 个迭代器 first 位置的值, 找到最小键值的迭代器, 让 current\_ 指向它。
###5.2 SeekToLast
逻辑同 SeekToFirst，先将 n\_ 个迭代器 seek 到 last 位置，找到最大键值的迭代器，让 current\_ 指向它。
###5.3 Seek(const Slice& target)
定位到首个键值大于 target 的位置。先将 n\_ 个迭代器 seek 到首次各自键值大于等于 target 的位置，再比较它们此时各自键值，
找到最小键值的迭代器，让 current\_ 指向它。
注意 leveldb 整个体系中，查找一个键值的逻辑均是：寻找到首个键值大于等于目标键的位置。本文 2.3 节中有指出这点。
###5.4 Next
这个接口的实现有些复杂。
MergingIterator 有如下设计：
- 1.当迭代器有相同的 InternalKey 时它们会被多次遍历到。
- 2.若 MergingIterator 中两个迭代器 x, y 的当前位置键值一样，则谁在 MergingIterator 中下标小则更小。否则更大。
- 3.“反方向遍历”的设计是，保证此次遍历之后的键值不等于当前键值。
- 4."Next 之后 Prev" 遍历的设计是，定位到比 MergingIterator当前键值小的最大那个位置。
- 5."Prev 之后 Next" 遍历的设计是，定位到比 MergingIterator当前键值大的最小那个位置。
- 6.由3，4两点可知，调用Next再调用Prev 不一定能回到之前的那个位置
1.如果上一次是向后遍历的（即与 Next 方向一致），现在调用 Next，我们需要计算 current\_ 所在迭代器的下一个位置的键值与其它
迭代器位置的键值中最小的那个，即是整个迭代器组的 Next 位置。
2.如果上一次是向前遍历的（与 Next 方向相反），现在调用 Next，则将所有除 current\_ 的分迭代器都 seek 到 current\_.key() 的位置。
再判断若某些分迭代器位置有效且的键值为 current\_.key() (即不大于 current\_.key())则对分迭代器调用 Next。再使用 FindSmallest 找出最小的那个分迭代器，让 current\_ 指向它。

###5.5 Prev
与 Next 相对应，
1.如果上一次是向前遍历的（即与 Prev 方向一致），现在调用 Prev，我们需要计算 current\_ 所在迭代器的上一个位置的键值与其它
迭代器位置的键值中最大的那个，即是整个迭代器组的 Prev 位置。
2.如果上一次是向后遍历的（与 Prev 方向相反），现在调用 Prev，则将所有除 current\_ 的迭代器都 seek 到 current\_.key() 的位置。
若分迭代器位置有效则调用对分迭代器 Prev。再使用 FindLargest 找出最大的那个迭代器，让 current\_ 指向它。

注意 Prev 与 Next 不同的地方在于，当它们都处于反方向遍历时，且都 seek 到了 current\_.key() 的位置，Next 中还需要判断各迭代器当前位置键
是否等于 current\_.key()，这是为了保证 MergingIterator 设计的第3条。所有迭代器的 seek(target) 的逻辑是，找到键值大于等于 target 的位置。若已经大于 current\_.key() 了则不需要对分迭代顺再调用 next 以满足第3条。

如下图x, y, z 表示 MergingIterator 的三个分迭代器，这三个迭代器各三个键(InternalKey)，三个分迭代器的键对应相等：它们的 user\_key 都一样，版本号分别为19，18，17，注意 InternalKey 的排序规则，版本号较大的排在前面(Prev 方向)。x18 表示 x 那个分迭代器的当前位置是18那个位置。红色三角符号表示 current\_ 指向的那个迭代器的当前位置。红色的线条把各个分迭代器的当前位置串联起来了。图演示了迭代器 Next 和 Prev 被调用时的状态变化。
![](/assets/leveldb/mergerIterator_1.JPG)

![](/assets/leveldb/mergerIterator_2.JPG)

[1]: 3003-leveldb精读系列-format
[2]: 3003-leveldb精读系列-key
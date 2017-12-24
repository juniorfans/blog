#TwoLevelIterator
TwoLevelIterator 是一个二维的迭代器. 即 TwoLevelIterator 的 value 是另一个迭代器对象的指针.
它的使用场景有两处：
- 1.生成某一层 sst 文件所有的键值对迭代器。
- 2.生成一个 Table 所有键值对迭代器。

我们来讨论它们的实现方式。
TwoLevelIterator 内部维持了两个迭代器：index_iter, data_iter。index_iter 是键与“低维迭代器构造信息”的对应，
data_iter 是键与值对应，它是实际遍历的对象。index_iter.value() 即是构造 data_iter 的输入。

##1.1 Seek(const Slice& target)
seek 的实现分三步
- 1.使用 index_iter.Seek(target) 定位到 target 对应的“低维迭代器构造信息”。
- 2.使用构造信息去生成一个 data_iter（可以优化为，若当前持有的 data_iter 正是要生成的则免去生成这一步）
- 3.调用 data_iter.Seek(target) 定位

##1.2 SeekToFirst
比较简单，index_iter.SeekToFirst，data_iter.SeekToFirst 即可。同 SeekToLast

##1.3 Next
对 TwoLevelIterator 当前持有的 data_iter.Next，若 data_iter 此时位置无效则需要 index_iter.Next，生成新的 data_iter，
更稳一点，确保是 data_iter 第一个元素，调用 data_iter.SeekToFirst。
实际上 TwoLevelIterator 的 Seek- 系列函数都作了此处理：若 data_iter 为 NULL 或当前位置无效则需要额外处理。详见代码注释。
同 Prev

##1.4 应用场景详解
本文最开始已经指出， TwoLevelIterator 有两个使用场景，它们各自的实现方式如下：
- 1.生成某一层 sst 文件所有的键值对迭代器。实现方式是：
	第一维的 index_iter 即是 LevelFileNumIterator，表示第 level 层文件列表的迭代器, 是键值与单个文件的映射. InternalKey --> (file number, file size)
	第二维的 data_iter 即是 TableCache 的 Iterator，它是 TableCache 上的键值对迭代器，使用1中的 (file number 和 file size)构造出来。
	
- 2.生成一个 Table 所有键值对迭代器。实现方式是：
	第一维的 index_iter 即是 sst 文件格式中的 index_block，表示这个 sst 文件的索引，是键值与 sst 中位置信息的映射. InternalKey --> (offset, size)
	第二维的 data_iter 是 Block 上的迭代器，用于返回元素的 key-value.

更多的细节见代码注释。

```
namespace {

typedef Iterator* (*BlockFunction)(void*, const ReadOptions&, const Slice&);


/************************************************************************/
/* 
lzh:	TwoLevelIterator 是一个二维的迭代器. 即 TwoLevelIterator 的 value 与另一个迭代器对象关联. 
		查找一个元素需要两次 Seek：第一次找到此元素所在的迭代器，第二次在上一个迭代器上查找此元素。
*/
/************************************************************************/
class TwoLevelIterator: public Iterator {
 public:
  TwoLevelIterator(
    Iterator* index_iter,
    BlockFunction block_function,
    void* arg,
    const ReadOptions& options);

  virtual ~TwoLevelIterator();

  virtual void Seek(const Slice& target);
  virtual void SeekToFirst();
  virtual void SeekToLast();
  virtual void Next();
  virtual void Prev();

  virtual bool Valid() const {
    return data_iter_.Valid();
  }
  virtual Slice key() const {
    assert(Valid());
    return data_iter_.key();
  }
  virtual Slice value() const {
    assert(Valid());
    return data_iter_.value();
  }
  virtual Status status() const {
    // It'd be nice if status() returned a const Status& instead of a Status
    if (!index_iter_.status().ok()) {
      return index_iter_.status();
    } else if (data_iter_.iter() != NULL && !data_iter_.status().ok()) {
      return data_iter_.status();
    } else {
      return status_;
    }
  }

 private:
  void SaveError(const Status& s) {
    if (status_.ok() && !s.ok()) status_ = s;
  }
  void SkipEmptyDataBlocksForward();
  void SkipEmptyDataBlocksBackward();
  void SetDataIterator(Iterator* data_iter);
  void InitDataBlock();

  BlockFunction block_function_;
  void* arg_;
  const ReadOptions options_;
  Status status_;
  IteratorWrapper index_iter_;
  IteratorWrapper data_iter_; // May be NULL
  // If data_iter_ is non-NULL, then "data_block_handle_" holds the
  // "index_value" passed to block_function_ to create the data_iter_.
  std::string data_block_handle_;
};

TwoLevelIterator::TwoLevelIterator(
    Iterator* index_iter,
    BlockFunction block_function,
    void* arg,
    const ReadOptions& options)
    : block_function_(block_function),
      arg_(arg),
      options_(options),
      index_iter_(index_iter),
      data_iter_(NULL) {
}

TwoLevelIterator::~TwoLevelIterator() {
}

void TwoLevelIterator::Seek(const Slice& target) {
  index_iter_.Seek(target);
  InitDataBlock();
  if (data_iter_.iter() != NULL) data_iter_.Seek(target);
  SkipEmptyDataBlocksForward();
}

void TwoLevelIterator::SeekToFirst() {
  index_iter_.SeekToFirst();
  InitDataBlock();
  if (data_iter_.iter() != NULL) data_iter_.SeekToFirst();
  SkipEmptyDataBlocksForward();
}

void TwoLevelIterator::SeekToLast() {
  index_iter_.SeekToLast();
  InitDataBlock();
  if (data_iter_.iter() != NULL) data_iter_.SeekToLast();
  SkipEmptyDataBlocksBackward();
}

void TwoLevelIterator::Next() {
  assert(Valid());
  data_iter_.Next();
  SkipEmptyDataBlocksForward();
}

void TwoLevelIterator::Prev() {
  assert(Valid());
  data_iter_.Prev();
  SkipEmptyDataBlocksBackward();
}

/************************************************************************/
/* 
	lzh: 向前(Next 方向)跳过空的或无效的数据块
*/
/************************************************************************/
void TwoLevelIterator::SkipEmptyDataBlocksForward() {
  while (data_iter_.iter() == NULL || !data_iter_.Valid()) {
    // Move to next block
    if (!index_iter_.Valid()) {
      SetDataIterator(NULL);
      return;
    }
	//lzh: 索引块往后遍历，即得到下一个数据块
    index_iter_.Next();
    InitDataBlock();
	//lzh: 新块的第一个位置
    if (data_iter_.iter() != NULL) data_iter_.SeekToFirst();
  }
}

/************************************************************************/
/* 
	lzh: 向后(Prev 方向)跳过空的或无效的数据块
*/
/************************************************************************/
void TwoLevelIterator::SkipEmptyDataBlocksBackward() {
  while (data_iter_.iter() == NULL || !data_iter_.Valid()) {
    // Move to next block
    if (!index_iter_.Valid()) {
      SetDataIterator(NULL);
      return;
    }
    index_iter_.Prev();
    InitDataBlock();
    if (data_iter_.iter() != NULL) data_iter_.SeekToLast();
  }
}

void TwoLevelIterator::SetDataIterator(Iterator* data_iter) {
  if (data_iter_.iter() != NULL) SaveError(data_iter_.status());
  data_iter_.Set(data_iter);
}


/************************************************************************/
/* 
	lzh: 此函数在该迭代器的所有 Seek 系列方法中被调用, 
	注意当前迭代器 TwoLevelIterator 的值是 data_iter_ 迭代器中的索引
*/
/************************************************************************/
void TwoLevelIterator::InitDataBlock() {
  if (!index_iter_.Valid()) {
    SetDataIterator(NULL);
  } else {
    Slice handle = index_iter_.value();
    if (data_iter_.iter() != NULL && handle.compare(data_block_handle_) == 0) {
      // data_iter_ is already constructed with this iterator, so
      // no need to change anything
    } else {

      Iterator* iter	//lzh: 
		  = 
		  (*block_function_)(
		  arg_,			//lzh: table 
		  options_,		//lzh: read option (block cache 挂在 option 中)
		  handle		//lzh: handle 即是元素在 talbe 中的位置信息
		  );
      data_block_handle_.assign(handle.data(), handle.size());
      SetDataIterator(iter);
    }
  }
}

}

Iterator* NewTwoLevelIterator(
    Iterator* index_iter,
    BlockFunction block_function,
    void* arg,
    const ReadOptions& options) {
  return new TwoLevelIterator(index_iter, block_function, arg, options);
}
```
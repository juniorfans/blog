#Block::Iter
这个迭代器用来遍历 Block，Block 的格式见 [3003-leveldb精读系列-format][1]，Block 中的 entry 是按 key 的顺序来排列的，这一点需要时时在心中铭记。
Block::Iter 迭代器的实现并不是基于在内存中构建 Block 已经解析好的 entry 链表或者数组，而是直接在字节流上建立起来的。维持着当前 Block 的字节流，当前 entry 字节流的偏移位置，重启点(每组起始位置)数组存储的起始位置，重启点个数，当前 entry 所在的组的下标。 
Block::Iter 实现如下：
##1.1 Next
- 1.通过当前 entry 的最后一个元素 value 的位置及长度，计算并移动到下一个 entry 的起始地址
- 2.调用 DecodeEntry 解析当前 entry 数据
- 3.重置迭代器中的位置， key, value, 当前 entry 所在组的下标 restart_index

##1.2 Prev
- 1.计算前面一个 entry 所在的那个组，原代码中使用了循环，对比每个组的首个 entry 地址与当前 entry 位置比较，找到首个地址小于 entry 位置的最大的组，即是前一个 entry 所在的组。但个人感觉不是必须的，这一点如果有人发现了原因烦请告之(1127365587@qq.com)。
- 2.使用 SeekToRestartPoint 将迭代器定位到第1步中找到的那个组的首地址
- 3.在这个组中循环向后找，解析每个 entry 并设置迭代器位置， key, value 等信息，直至遇到当前 entry 即停止。此时最后一个被解析即是前一个 entry。详见代码中的 do while

##1.3 Seek(const Sliece& target)
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

[1]: 3003-leveldb精读系列-format
[2]: 3003-leveldb精读系列-key



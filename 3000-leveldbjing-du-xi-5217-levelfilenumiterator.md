#LevelFileNumIterator
这个迭代器用于遍历某层的 sst 文件。迭代器维持了一个指向文件的下标 index_，通过它的增减表示向前后的遍历。
index_ 等于文件总个数时表示迭代器失效。
下面是迭代器主要方法的实现
##1.1 Next
判断当前迭代器有效性。直接增加 index_
##1.2 Prev
判断当前迭代器有效性。若当前 index_ 为0则设置为文件个数，表示失效。直接减小 index_
##1.3 Seek(const Slice& target)
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

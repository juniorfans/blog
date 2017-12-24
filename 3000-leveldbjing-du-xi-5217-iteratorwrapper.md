#IteratorWrapper
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




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

##1.7 代码注释

```
/************************************************************************/
/* 
	lzh: MergingIterator 对外提供 Prev, Next, Seek 接口, 
		对内的实现是管理着一堆 IteratorWrapper, 通过它们的 Prev, Next, Seek 实现那些功能

		称 MergingIterator 管理的多个迭代器为“分迭代器”. 若本次遍历的方向与上次不一样则称为“反向遍历”，如
		上一次是 Next 本次是 Prev，或者上一次是 Prev 本次是 Next

		MergingIterator 的实现有如下规则
		- 1.当多个分迭代器有相同的 InternalKey 时，这些键会被多次遍历到。
		- 2.若 MergingIterator 中两个迭代器 x, y 的当前位置键值一样，则谁在 MergingIterator 中下标小则更小。否则更大。
		- 3.“反方向遍历”的设计是，保证当次遍历的键值不等于上一次的键值。
		- 4."Next 之后 Prev" 遍历的设计是，定位到比 MergingIterator当前键值小的最大那个位置。
		- 5."Prev 之后 Next" 遍历的设计是，定位到比 MergingIterator当前键值大的最小那个位置。
		- 6.由3，4两点可知，调用Next再调用Prev 不一定能回到之前的那个位置
*/
/************************************************************************/
class MergingIterator : public Iterator {
 public:
  MergingIterator(const Comparator* comparator, Iterator** children, int n)
      : comparator_(comparator),
        children_(new IteratorWrapper[n]),
        n_(n),
        current_(NULL),
        direction_(kForward) {
    for (int i = 0; i < n; i++) {
      children_[i].Set(children[i]);
    }
  }

  virtual ~MergingIterator() {
    delete[] children_;
  }

  virtual bool Valid() const {
    return (current_ != NULL);
  }

  /************************************************************************/
  /* 
	lzh: 迭代器 seek 到 first 位置: 需要将那一堆 WrapperIterator 全部 seek 到 first 位置
	并且通过比较这 n_ 个迭代器 first 位置的值, 找到最小的迭代器, 让 current_ 指向它
  */
  /************************************************************************/
  virtual void SeekToFirst() {
    for (int i = 0; i < n_; i++) {
      children_[i].SeekToFirst();
    }
    FindSmallest();
    direction_ = kForward;
  }

  //lzh: 同 SeektoFirst
  virtual void SeekToLast() {
    for (int i = 0; i < n_; i++) {
      children_[i].SeekToLast();
    }
    FindLargest();
    direction_ = kReverse;
  }

  /************************************************************************/
  /* 
	lzh: 在各个迭代器中 seek 到 target 的位置, 然后比较这 n_ 个位置的值, 
	找到最小的, 让 current_ 指向那个迭代器
  */
  /************************************************************************/
  virtual void Seek(const Slice& target) {
    for (int i = 0; i < n_; i++) {
      children_[i].Seek(target);
    }
    FindSmallest();
    direction_ = kForward;
  }

  /************************************************************************/
  /* 
	lzh: 分两种情况
		1.	若上一次遍历的方向是 kForward, 即与本函数需要的遍历方向
			一致, 则只需要将 迭代器 current_ 下一个元素 next 与其它迭代器各自指
			的位置的值进行比较, 找到最小的, 即是整组迭代器的 Next.
		2.	若上一次遍历的方向是 kReverse, 与本次要遍历方向相反, 当前键值是 current_.key()，
			则 Next 的意图是要定位到大于此键的最小键的那个位置。因此实现的方式是
			a).非 current_ 的分迭代器中 seek 到键值首次大于 current_.key() 的位置（使用 Seek 函数定位时注意如果键
			值刚好等于 current_.key() 则需要再对该分迭代器调用 Next，以保证"大于"）。
			b).current_ 直接 Next
  */
  /************************************************************************/
  virtual void Next() {
    assert(Valid());

    // Ensure that all children are positioned after key().
    // If we are moving in the forward direction, it is already
    // true for all of the non-current_ children since current_ is
    // the smallest child and key() == current_->key().  Otherwise,
    // we explicitly position the non-current_ children.
    if (direction_ != kForward) {
      for (int i = 0; i < n_; i++) {
        IteratorWrapper* child = &children_[i];
        if (child != current_) {
          child->Seek(key());
          if (child->Valid() &&
              comparator_->Compare(key(), child->key()) == 0) {
            child->Next();
          }
        }
      }
      direction_ = kForward;
    }

    current_->Next();
    FindSmallest();
  }

  /************************************************************************/
  /* 
  lzh: 与 Next 类似, 分两种情况
	  1.	若上一次遍历的方向是 kReverse, 即与本函数需要的遍历方向
		  一致, 则只需要将 迭代器 current_ 下一个元素 prev 与其它迭代器各自指
		  的位置的值进行比较, 找到最大的, 即是整组迭代器的 Prev.
	  2.	若上一次遍历的方向是 kForward, 与本次要遍历方向相反, 当前键值是 current_.key()，
			则 Prev 的意图是要定位到小于此键的最大键的那个位置，实现的方式是:
			a).非 current_ 迭代器调用 Seek 定位到首个大于等于 current_.key() 的位置，再对它们都调用 Prev，
				保证这些迭代器所处的位置是小于 current_.key() 的最大位置
			b).current_ 迭代器直接 Prev
  */
  /************************************************************************/
  virtual void Prev() {
    assert(Valid());

    // Ensure that all children are positioned before key().
    // If we are moving in the reverse direction, it is already
    // true for all of the non-current_ children since current_ is
    // the largest child and key() == current_->key().  Otherwise,
    // we explicitly position the non-current_ children.
    if (direction_ != kReverse) {
      for (int i = 0; i < n_; i++) {
        IteratorWrapper* child = &children_[i];
        if (child != current_) {
          child->Seek(key());
          if (child->Valid()) {
            // Child is at first entry >= key().  Step back one to be < key()
            child->Prev();
          } else {
            // Child has no entries >= key().  Position at last entry.
            child->SeekToLast();
          }
        }
      }
      direction_ = kReverse;
    }

    current_->Prev();
	//lzh: 
    FindLargest();
  }

  virtual Slice key() const {
    assert(Valid());
    return current_->key();
  }

  virtual Slice value() const {
    assert(Valid());
    return current_->value();
  }

  virtual Status status() const {
    Status status;
    for (int i = 0; i < n_; i++) {
      status = children_[i].status();
      if (!status.ok()) {
        break;
      }
    }
    return status;
  }

 private:
  void FindSmallest();
  void FindLargest();

  // We might want to use a heap in case there are lots of children.
  // For now we use a simple array since we expect a very small number
  // of children in leveldb.
  const Comparator* comparator_;

  //lzh: 共计 n_ 个 IteratorWrapper 对象, children_[i] 指第 i 个 IteratorWrapper 对象
  IteratorWrapper* children_;	
  int n_;

  //lzh: 指向当前正在被使用的那个 IteratorWrapper 对象的指针
  IteratorWrapper* current_;

  // Which direction is the iterator moving?
  enum Direction {
    kForward,
    kReverse
  };
  Direction direction_;
};

/************************************************************************/
/* 
	lzh: 正向遍历，定位到最小的。
	当第 x 个与第 y 个迭代器的 key 一样时，x< y，返回 x。这一点与 FindSmallest 相反。
	这是因为 FindSmallest 与 Next 方向一致：相等的 key ，先遇到的比较小
*/
/************************************************************************/
void MergingIterator::FindSmallest() {
  IteratorWrapper* smallest = NULL;
  for (int i = 0; i < n_; i++) {
    IteratorWrapper* child = &children_[i];
    if (child->Valid()) {
      if (smallest == NULL) {
        smallest = child;
	  } else {
			Slice childKey = child->key();
			Slice smallestKey = smallest->key();
			int comRes = comparator_->Compare(child->key(), smallest->key());

			Slice childValue = child->value();
			Slice smallestValue = smallest->value();
			InternalKey ck, sk;
			ck.DecodeFrom(childKey);
			sk.DecodeFrom(smallestKey);

		//	printf("childKey: %s (%s), smallestKey: %s (%s), compareRes: %d\r\n", \
				ck.user_key().ToString().c_str(), childValue.ToString().c_str(),\
				sk.user_key().ToString().c_str(), smallestValue.ToString().c_str(),\
				comRes\
				);

		  if (comparator_->Compare(child->key(), smallest->key()) < 0) {
			  smallest = child;
			}
	  }
    }
  }
  current_ = smallest;
}

/************************************************************************/
/* 
	lzh: 反向遍历，定位到最大的，
	当第 x 个与第 y 个迭代器的 key 一样时，x > y，返回 x。这一点与 FindSmallest 相反。
	先 Findsmallest 逻辑保持一致，相同的 key 先遇到的较小，所以后遇到的较大。
*/
/************************************************************************/
void MergingIterator::FindLargest() {
  IteratorWrapper* largest = NULL;
  for (int i = n_-1; i >= 0; i--) {
    IteratorWrapper* child = &children_[i];
    if (child->Valid()) {
      if (largest == NULL) {
        largest = child;
      } else if (comparator_->Compare(child->key(), largest->key()) > 0) {
        largest = child;
      }
    }
  }
  current_ = largest;
}
}

/************************************************************************/
/* 
	lzh: 构建一个 MergeIterator，注意下面的 cmp 是 InternalKeyComparator
*/
/************************************************************************/
Iterator* NewMergingIterator(const Comparator* cmp, Iterator** list, int n) {
  assert(n >= 0);
  if (n == 0) {
    return NewEmptyIterator();
  } else if (n == 1) {
    return list[0];
  } else {
    return new MergingIterator(cmp, list, n);
  }
```
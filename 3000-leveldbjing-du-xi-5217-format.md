#leveldb 中的 format
##1 概述
本文讲述 leveldb 中文件层次（多级 sst + memtalbe），sst 文件格式，log 文件格式，Block 格式，比较器 comparetor。

##2 leveldb Block 格式
leveldb 中最细粒度的是 Key，包括 userkey，InternalKey，ParsedInternalKey，详见[“leveldb精读系列-key”][1]，此文介绍了它们各自的定义，以及它们存在的意义。最终存储的形式就是 Key-Value 的形式。为了减少存储空间，使用了 Block，它是 Key-Value 的容器：每个 Block 有多组 Key-Value。
为了提高查询效率和减少存储空间，Block 有如下设计：
- 1.每个 Block 中还存储着它里面所有 Key 的范围以便于快速决定一个键是否在它里面
- 2.每个 Block 中的键值对是有序排列的，于是可以利用这点，将相领的 Key 共用的前缀串压缩存储。具体的算法是：每 **block_restart_interval** 个键值对作为一个组，第2个到最后一个 Key-Value 在存储时，只存储各个 Key 与第一个键值对 Key 的相同前缀长度和差异部分。

下图展示的是 block 的格式。
![](/assets/leveldb/sst_block.bmp)

###2.1 Block.entry
每个 Block 有多个 entry，每个 entry 就是一个 Key-Value：shared_bytes 表示在一组 entry 里面，当前 entry 的 Key 与第一个 entry 的 Key 相同的前缀字节数。unshard\_bytes 即是当前 entry 独有的字节数，value\_bytes 即是当前 entry 的值的长度，unshared\_key\_data 是当前 entry 独有的 key 内容，value_data 即是值内容。从 entry 的定义来看，存储了 key 和 value 的长度及字节序列，它可以存储任意的字节流，哪怕是图片，这一点很棒！
###2.2 Block
Block 包括多组 entry。各组是一个独立“共用前缀存储”的单元。restarts 是一个数组，存储着各组第一个元素的字节位置。num\_of\_restarts 是 restarts 的元素个数。

更多的细节见于代码注释。

```
//block_builder.cc 文件
BlockBuilder::BlockBuilder(const Options* options)
    : options_(options),
      restarts_(),
      counter_(0),
      finished_(false) {
  assert(options->block_restart_interval >= 1);
  restarts_.push_back(0);       // First restart point is at offset 0
}

void BlockBuilder::Reset() {
  buffer_.clear();
  restarts_.clear();
  restarts_.push_back(0);       // First restart point is at offset 0
  counter_ = 0;
  finished_ = false;
  last_key_.clear();
}

size_t BlockBuilder::CurrentSizeEstimate() const {
  return (buffer_.size() +                        // Raw data buffer
          restarts_.size() * sizeof(uint32_t) +   // Restart array
          sizeof(uint32_t));                      // Restart array length
}

Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}

/************************************************************************/
/* 
	lzh: 加入 (key, value) 到 block 中 ->
		1. 一个 block 包含多个 entry
		2. 一个 entry 包含一个 (key, value), 但是为了压缩, 后面的 entry 与前面的 entry 采用 diff 方式记录.
		3. 具体而言: leveldb对key的存储进行前缀压缩，每个entry中会记录key与前一个key前缀相同的字节（shared_bytes）以及自己独有的字节（unshared_bytes）
			读取时，对block进行遍历，每个key根据前一个key以及shared_bytes/unshared_bytes可以构造出来
		4. 按 block 内的 entry 完全采用 diff 方式记录, 则像链表一样: 要随机地解析任意一个 entry 都需要从第一个开始, 所以优化为: 每 block_restart_interval 
			个 entry 采用 diff 方式记录.
		
		5. 由于 Add 进来的 key 是按递增顺序加入的, 所以后一个 key 与前一个有较高的重叠概率

		6. block 最后的五个字节(1字节表示是否压缩后4字节为 crc 校验码)是在完成一个 block 时写入的. 在 table_build.cc 中

	lzh: 结构图详见本工程的资源文件: sst_and_block.bmp
*/
/************************************************************************/
void BlockBuilder::Add(const Slice& key, const Slice& value) {
	//lzh: last_key  即是上一个 key
  Slice last_key_piece(last_key_);
  assert(!finished_);

  //lzh: counter_ 一旦为 block_restart_interval 将会重新开始新一轮, 被设置为 0
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty() // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;

  //lzh: 当前处于一个 diff 记录中
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    const size_t min_length = std::min(last_key_piece.size(), key.size());

	//lzh: last_key_piece 指上一个 entry 的 key. 计算当前的 key 与它有多长的重叠
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
	  //lzh: 完成一个区间的 diff 记录, 记录当前一轮的 entry 的大小
    // Restart compression
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  //lzh: buffer_ 的格式为: 相同的字节数 | 不同的字节数 | value 字节数 | key 中不同的内容 | value 内容
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
  //lzh: 下面两步计算完全是为了验证 last_key_ 是否和 key 一样
  last_key_.resize(shared);	//lzh: last_key_ 此时的值是上一个 key, 而 last_key_ 的前 shared 个字节与 key 的前 shared 个字节是一样的
  last_key_.append(key.data() + shared, non_shared);	//lzh: 将 key 的后面 non_shared 字节附到 last_key_ 后面, 
  assert(Slice(last_key_) == key);		//lzh: last_key 将与 key 一样
  counter_++;
}
```

##3 leveldb sst 格式
sstable 是更高层次的存储，它组合了多个 Block，同时维持着它们的索引信息，结构如下图：

![](/assets/leveldb/sst.bmp)

sst 中多个 Block 是 Key 有序的。
meta\_block 对应于一个 data\_block，保存data\_block中的key size/value size/kv counts之类的统计信息，当前版本未实现。
metaindex\_block: 保存meta\_block的索引信息。当前版本未实现。
index\_block: 每个 data\_block 的 offset/size 都会写入到这个 index\_block 中，起到索引的作用。
footer: 文件末尾的固定长度的数据。保存着metaindex\_block和index\_block的索引信息, 为达到固定的长度，添加padding_bytes。最后有8个字节的magic校验。

##4 leveldb 文件层次
leveldb 存储数据的形式是 sst 文件，为了查找/写入数据的效率，对这些文件分总共7层进行管理（第0到6层）。第1层到第6层：每层的文件



##5 leveldb 限制
###3.1 第0层文件的限制
leveldb::config 中定义了第 0 层的文件有如下限制
- 1.leveldb 多层存储结构最多 7 层
- 2.第 0 层的 sst 文件在达到 4 个即触发 compaction
- 3.第 0 层的 sst 文件达到 8 个时就会降低写入的速度
- 4.第 0 层的 sst 文件达到 12 个时会暂停写入

###3.2 memtable 的限制
还定义了 memtalbe 在 dump 到磁盘上的限制：memtable 最高作为第 2 层的 sst 写到磁盘。

```
//block_builder.cc
namespace config {
static const int kNumLevels = 7;

// Level-0 compaction is started when we hit this many files.
//lzh: 第 0 层文件最多4个文件，到达此限制则开始 compaction
static const int kL0_CompactionTrigger = 4;

// Soft limit on number of level-0 files.  We slow down writes at this point.
//lzh: 第 0 层文件到达了 8 个文件时，则开始限制写入的速度
static const int kL0_SlowdownWritesTrigger = 8;

// Maximum number of level-0 files.  We stop writes at this point.
//lzh: 第 0 层文件到达了 12 个文件时开始暂停写入
static const int kL0_StopWritesTrigger = 12;

// Maximum level to which a new compacted memtable is pushed if it
// does not create overlap.  We try to push to level 2 to avoid the
// relatively expensive level 0=>1 compactions and to avoid some
// expensive manifest file operations.  We do not push all the way to
// the largest level since that can generate a lot of wasted disk
// space if the same key space is being repeatedly overwritten.
//lzh: 在将 memtable 写到磁盘上时，并非一定会写到第 0 层，因为这可能导致
//lzh: 第 0 层文件与第 1 层文件有较大的重叠，增加后面 compaction 负担，
//lzh: 所以可以跳过一些层次，直接写到更高层次的文件中。
//lzh: kMaxMemCompactLevel 就指示了，从 memtable 写入到最高什么层次
static const int kMaxMemCompactLevel = 2;

}
```







[1]: leveldb精读系列-key
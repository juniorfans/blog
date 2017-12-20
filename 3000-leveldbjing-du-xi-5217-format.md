#leveldb 中的 format
##1 概述
本文讲述 leveldb 中文件层次（多级 sst + memtalbe），sst 文件格式，log 文件格式，Block 格式，比较器 comparetor。

##2 leveldb Block 格式
leveldb 中最细粒度的是 Key，包括 userkey，InternalKey，ParsedInternalKey，详见[“leveldb精读系列-key”][1]，此文介绍了它们各自的定义，以及它们存在的意义。最终存储的形式就是 Key-Value 的形式。为了减少存储空间，使用了 Block：每个 Block 有多组 Key-Value。
为了提高查询效率和减少存储空间，Block 有如下设计：
- 1.每个 Block 中存储着它里面所有 Key 的范围以便于快速决定一个键是否在它里面
- 2.每个 Block 中的键值对是有序排列的，于是可以利用这点，将相邻的 Key 共用的前缀串压缩存储。具体的算法是：每 **block_restart_interval** 个键值对作为一个组，***此组中每一个key在存储时，只存储它与前一个 Key 相同前缀的长度和差异部分***。采用分组的方式是为了提高随机解析的效率：解析第 n 个时只需要从它所在的组第一个开始解析，而不是从整个Block 第一个开始。

下图展示的是 block 的格式。
![](/assets/leveldb/sst_block.bmp)

###2.1 Block.entry
每个 Block 有多个 entry，每个 entry 就是一个 Key-Value：
- 1.shared_unshard\_bytes 即是当前 entry 独有的字节数，value\_bytes 即是当前 entry 的值的长度
- 2.unshared\_key\_data 是当前 entry 独有的 key 内容，value_data 即是值内容

从 entry 的定义来看，存储了 key 和 value 的长度及字节序列，它可以存储任意的字节流，哪怕是图片，这一点很棒！
###2.2 Block
Block 包括多组 entry。各组是一个独立“共用前缀存储”的单元。restarts 是一个数组，存储着各组的开始位置。num\_of\_restarts 是 restarts 的元素个数。

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

***sst 中多个 Block 是 Key 有序的***。
- 1.meta\_block 对应于一个 data\_block，保存data\_block中的key size/value size/kv counts之类的统计信息，当前版本未实现。
- 2.metaindex\_block: 保存meta\_block的索引信息。当前版本未实现。
- 3.index\_block: 每个 data\_block 的 offset/size 都会写入到这个 index\_block 中，起到索引的作用。
- 4.footer: 文件末尾的固定长度的数据。保存着metaindex\_block和index\_block的索引信息（位置和block内容大小）, 为达到固定的长度，添加padding_bytes。最后有8个字节的magic校验。 代码如下：

```
/************************************************************************/
/* 
	lzh: 将 footer 编码到 dst 中, 包含 metaindex 数据和 index 数据, 对齐后再加入 8 字节 magic
*/
/************************************************************************/
void Footer::EncodeTo(std::string* dst) const {
#ifndef NDEBUG
  const size_t original_size = dst->size();
#endif
  
  //压入 metaindex_handle_ 和 index_handle_, 也即压入 meta 和 index 的 offset/size 信息 (这两个都是变长编码的)
  metaindex_handle_.EncodeTo(dst);
  index_handle_.EncodeTo(dst);
  dst->resize(2 * BlockHandle::kMaxEncodedLength);  // Padding	//lzh: 对齐到 2 * BlockHandle::kMaxEncodedLength

  //lzh: 再压入 8 字节的 magic 数据
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber & 0xffffffffu));
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber >> 32));

  //lzh: 校验总长度
  assert(dst->size() == original_size + kEncodedLength);
}
```

所有生成 sst 的细节都在 table_builder.cc 中，我们全部放在这儿

```
namespace leveldb {

struct TableBuilder::Rep {
  Options options;
  Options index_block_options;
  WritableFile* file;
  uint64_t offset;
  Status status;
  BlockBuilder data_block;
  BlockBuilder index_block;
  std::string last_key;
  int64_t num_entries;
  bool closed;          // Either Finish() or Abandon() has been called.

  // We do not emit the index entry for a block until we have seen the
  // first key for the next data block.  This allows us to use shorter
  // keys in the index block.  For example, consider a block boundary
  // between the keys "the quick brown fox" and "the who".  We can use
  // "the r" as the key for the index block entry since it is >= all
  // entries in the first block and < all entries in subsequent
  // blocks.
  //

  // Invariant: r->pending_index_entry is true only if data_block is empty.
  //lzh: 有一个 data_block 刚刚完成了 Flush (此时 pending_index_entry 被设置为 true)，现在需要为它生成一个索引条目 
  bool pending_index_entry;

  //lzh: 生成那个完成了 Flush 的 data_block 对应的索引条目并加入到 index_block 中
  BlockHandle pending_handle;  // Handle to add to index block	

  std::string compressed_output;

  Rep(const Options& opt, WritableFile* f)
      : options(opt),
        index_block_options(opt),
        file(f),
        offset(0),
        data_block(&options),
        index_block(&index_block_options),
        num_entries(0),
        closed(false),
        pending_index_entry(false) {
    index_block_options.block_restart_interval = 1;
  }
};

TableBuilder::TableBuilder(const Options& options, WritableFile* file)
    : rep_(new Rep(options, file)) {
}

TableBuilder::~TableBuilder() {
  assert(rep_->closed);  // Catch errors where caller forgot to call Finish()
  delete rep_;
}

Status TableBuilder::ChangeOptions(const Options& options) {
  // Note: if more fields are added to Options, update
  // this function to catch changes that should not be allowed to
  // change in the middle of building a Table.
  if (options.comparator != rep_->options.comparator) {
    return Status::InvalidArgument("changing comparator while building table");
  }

  // Note that any live BlockBuilders point to rep_->options and therefore
  // will automatically pick up the updated options.
  rep_->options = options;
  rep_->index_block_options = options;
  rep_->index_block_options.block_restart_interval = 1;
  return Status::OK();
}

/************************************************************************/
/* 
	lzh: 往 sst 当前处理的块/Block 中加入 kv，若此块满了则 Flush(写入到磁盘)，
	且设置此块的偏移和大小, 生成此块的索引条目加入到 index_block
*/
/************************************************************************/
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;

  //lzh: 注意因为  r->last_key 可能是 r->num_entries 为零时通过 FindShortestSeparator 计算出来的, 所以要判断 r->num_entries
  if (r->num_entries > 0) {
	  //lzh: 保证 TableBuilder 以递增的顺序 Add
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  //lzh: pending_index_entry 的意思是: 当前块 r->data_block 满了, 已经 Flush 掉(写入磁盘), pending_index_entry 被设置为 true，当前需要写入那个块的 index 数据
  //lzh: 现在将刚刚已经 Flush 的 Block 的 index 信息写入 index_block
  if (r->pending_index_entry) {
    assert(r->data_block.empty());//lzh: Flush 完一个 data_block 后立即生成一个 index_block. 
									//lzh: 暂时停止生成 data_block 所以这里断定 data_block 为空

	//lzh: index_block 的每个条目是一个键值对: key -> index_handle。定位 k 所在的 Block 时，在 index_block 里面使用二分法
	//lzh: 搜索到最后一个键比 k 小的条目，再取出 index_handle，这样就知道了目标 Block 的偏移和大小。

	//lzh: r->last_key 是刚 Flush 的 Block 的最后一个键。函数参数 key 是当前要加入到下一个 Block 中的首个键。
	//lzh: 我们寻找比 r->last_key 大但比 key 小的最小值(长度比较短) 设置到 r->last_key 中
	//lzh: 作为 index_block 条目的键，这样可以在逻辑正确的情况下减少 index_block 占用的空间
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;

	//lzh: r->pending_handle 中的 offset/size 会在 WriteBlock 中被设置
    r->pending_handle.EncodeTo(&handle_encoding);

    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();

  //lzh: 当前块 data_block 的大小到达阈值, 执行 Flush.
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}

/************************************************************************/
/* 
	lzh: 
		1.将当前 data_block 写入到磁盘
		2.设置 r->pending_index_entry=true, 并记录当前块的偏移和大小, 下一次需要添加此块的 index 条目到 index_block
*/
/************************************************************************/
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;

  //lzh: 当前块还没有执行 Flush, 此块的 pending_index_entry 必然还是 false
  assert(!r->pending_index_entry);
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    r->pending_index_entry = true;	//lzh: 写入块完成后, 后续需要收集此块的 index 数据(在 TableBuilder::Add 做)
    r->status = r->file->Flush();
  }
}

/************************************************************************/
/* 
	lzh: 
		1.将 block 写入到磁盘(可能会压缩)
		2.将 block 的偏移大小写入到 handle 中，（注意首个 block 的 offset是0，且 block 的 size 不含 trailer 大小）
*/
/************************************************************************/
void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  Slice raw = block->Finish();

  Slice block_contents;
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    case kSnappyCompression: {
      std::string* compressed = &r->compressed_output;
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        block_contents = *compressed;
      } else {
        // Snappy not supported, or compressed less than 12.5%, so just
        // store uncompressed form
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }
  }

  //lzh: 记录当前块的 index 数据. 
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());

  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {

	  //lzh: 每个block后面都会有5个字节的trailer，第1个字节表示 block 内的数据是否采用了压缩, 后面 4个字节是 block 数据的 crc 校验码. 参见 资源文件 sst_and_block.bmp 文件
    char trailer[kBlockTrailerSize];
    trailer[0] = type;
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    EncodeFixed32(trailer+1, crc32c::Mask(crc));
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      r->offset += block_contents.size() + kBlockTrailerSize;	//下一个 block 的 offset 应该是上个块的 offset 加上上个块的大小和 trailer 大小
    }
  }
  r->compressed_output.clear();
  block->Reset();
}

Status TableBuilder::status() const {
  return rep_->status;
}

/************************************************************************/
/* 
	lzh: 最后调用 Finish 完成这些事情：
		1.Flush 掉还没有写入磁盘的 data_block
		2.写入一个 metaindex_block，当前的实现是直接写入一个空的块
		3.生成最后一个 Flush 的块的索引条目并加入到 index_block 中
		4.将 index_block 写入到磁盘

		注意当前版本没有写入 meta_block

	回顾一下 sst 的格式:
	1. meta_block：每个data_block对应一个meta_block ，保存data_block中的key size/value size/kv counts之类的统计信息，当前版本未实现
	2. metaindex_block: 保存meta_block的索引信息。当前版本未实现。
	3. index_block: 每个 data_block 的 offset/size 都会写入到这个 index_block 中
	4. footer: 文件末尾的固定长度的数据。保存着metaindex_block和index_block的索引信息, 为达到固定的长度，添加padding_bytes。最后有8个字节的magic校验
*/
/************************************************************************/
Status TableBuilder::Finish() {
  Rep* r = rep_;
  //lzh: 当前块还没有处理完，写入磁盘
  Flush();
  assert(!r->closed);
  r->closed = true;
  BlockHandle metaindex_block_handle;
  BlockHandle index_block_handle;
  if (ok()) {
	  //lzh: 下面写入一个空的 block 
    BlockBuilder meta_index_block(&r->options);
    // TODO(postrelease): Add stats and other meta blocks
	//lzh: 因为没有对 meta_index_block 调用过 Add  函数，这个 block 中没有 entry
	//lzh: 此处写入一个空的 block, 只含有 num_of_restarts(0) 及 trailer
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }
  if (ok()) {
	  //lzh: 最后一个处理的块的 index 还没有加入到 index_block 里，现在处理
    if (r->pending_index_entry) {

		//lzh: 最后一个块, 找到大于 r->last_key 的最小值. 目的同 FindShortestSeparator, 减少 index_block 中键的长度
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }

	//lzh: 接下来写入 index_block
    WriteBlock(&r->index_block, &index_block_handle);
  }
  //lzh: 最后写入 footer: metaindex_block 的偏移大小和 index_block 的偏移大小
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;

	//lzh: 压入 metaindex_block_handle 和 index_block_handle (对齐后再加入 magic)
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}

void TableBuilder::Abandon() {
  Rep* r = rep_;
  assert(!r->closed);
  r->closed = true;
}

uint64_t TableBuilder::NumEntries() const {
  return rep_->num_entries;
}

uint64_t TableBuilder::FileSize() const {
  return rep_->offset;
}

}

```


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
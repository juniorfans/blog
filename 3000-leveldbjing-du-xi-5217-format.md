#leveldb 中的 format
##1 概述
本文讲述 leveldb 中文件层次（多级 sst + memtalbe），sst 文件格式，log 文件格式，Block 格式，比较器 comparetor。
##2 leveldb Block 格式

##2 leveldb 文件层次
leveldb 存储数据的形式是 sst 文件，为了查找/写入数据的效率，对这些文件分总共7层进行管理（第0到6层）。第1层到第6层：每层的文件



##3 leveldb 限制
###3.1 第0层文件的限制
leveldb::config 中定义了第 0 层的文件有如下限制
- 1.leveldb 多层存储结构最多 7 层
- 2.第 0 层的 sst 文件在达到 4 个即触发 compaction
- 3.第 0 层的 sst 文件达到 8 个时就会降低写入的速度
- 4.第 0 层的 sst 文件达到 12 个时会暂停写入

###3.2 memtable 的限制
还定义了 memtalbe 在 dump 到磁盘上的限制：memtable 最高作为第 2 层的 sst 写到磁盘。

```
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

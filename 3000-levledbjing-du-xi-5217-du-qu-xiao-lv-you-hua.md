# leveldb 读取效率优化

##0 iterator 顺序访问

###0.0 内部迭代器

DB::NewIterator 每次调用会生成以下迭代器：
1.mem\_->NewIterator
2.imm\_->NewIterator
3.每个第 0 层文件构成一个迭代器
4.从第1层开始，每层文件构成一个迭代器

###0.0 迭代器访存

- mem\_ 和 imm\_ 皆是内存中的 skiplist，整个数据库只有这两个，生成 iterator 并不会有复制器出来。
- 每个第 0 层的文件构成的迭代器，均放在 Version 对象的 table\_cache\_ 中：若某个 sst 文件不在内存中，则读取 sst 文件的 footer 和 index\_block 到内存(注意，此时并未读取 data\_block )。后续在查找时，若某个 Block 不在内存中，则加到全局的 block 缓存器中：options 中定义的：Cache* block\_cache;，默认的大小是 8<<20 即 8 M。
- 从第 1 层开始的每层一个迭代器，也是依托在 table\_cache\_ 的：每层是一个 TwoLevelIterator，先通过第一个迭代器， 定位 user\_key 到某层某文件，再通过第二个迭代器，定位某层某文件所在的位置(如缓存的位置)，最后，在那个位置查找 user\_key 的值。
- 整个数据库只有一个  Version，从而也只有一个 table\_cache。

### 0.1 table\_cache 的组织方式是：

- 1.使用一个 LRUCahce 对象控制和缓存打开的文件信息(table\_cache.cc 中 ,默认最多只能 1000-10，即 990 个)
- 2.使用一个 LRUCache 缓存了所有打开的 Block(table.cc 中 BlockReader 函数)

LRUCache 的行为是，淘汰最近最久未使用的缓存。

由以上可知，关键点是：**当有多个线程同时进行顺序访问时，由于 block\_cache 较小，全局唯一的  block\_cache 会经常被换入换出，导致性能降低**。
优化策略：**若每个 block 块大小为 2M，当有 N 个线程时，比较适当的一种设置方式是，block\_cache 大小应该为至少 N*2 兆比较合适：这可以避免某个线程加载了 2M 数据到缓存，不会立即被换出**。
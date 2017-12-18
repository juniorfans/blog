#leveldb 中的 builder
##概述
本文讲述 leveldb 中 TableBuilder，BlockBuilder。它们分别展示了，如何使用一个迭代器构建出一个 Table，如何通过一次次加入数据构建出一个 Block。
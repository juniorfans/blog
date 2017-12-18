#leveldb 的 Version
##概述
本文包括 leveldb 中的 Version 管理(VersionSet，链表式管理)，如何管理 db 中所有的 sst 文件，根据 key 的范围查找文件，查找 key 重叠的文件，如何使用 LevelFileNumIterator 去遍历 Version 中所有文件，如何遍历某一层的文件，提供查询接口，管理引用，如何应用(Apply)一个 version，何时更改 Compation 的信息，如何 Recover，
如何构建出快照功能 snopshot，等等
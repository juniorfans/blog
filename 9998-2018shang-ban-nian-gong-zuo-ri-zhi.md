# 工作日志
## 1.golang 安放依赖包
先运行 go get -u -ldflags -H=windowsgui github.com/nsf/gocode 
会在 GOPATH 下建立 src/github.com/nsf/gocode 文件夹，在无法访问外网情况下，目录为空。离线下载后将文件放入此目录，再运行 go install 即可安装  gocode

## 2.golang 编译
go build 后面可以接文件, 此时，从当前目录出发(从 GoPath 出发不作数)，一定要能保证找得到那个文件。
go build 后面可以接目录名(看起来像是包名但其实是目录). 从当前目录出发，或者 GoPath 下有这个目录名即可。
go install 与 go build 相同，也可以接文件和目录名，与 go build 规则一样。
当位于一个目录下时，可以直接使用 go build ，不带参数，表示编译此目录下所有的文件。
当一个目录下存在引入不同包的 go 文件，使用 go build 会报错。
注意，当 go build/install 的参数是目录时，build/install 的逻辑是非递归的，它只会处理所指目录下的文件，它里面的目录不会递归处理。
补充一条规则：当 go build/install  接目录时，而当前 shell 所处的目录不在 GOPATH 里面，则上面的命令报错，因为 go 命令是全局式的，它找目录的方式永远是从 GOPATH 开始找。
GOPATH 下的目录下的 src 目录可以被自动忽略，这是因为， go 语言假设所有工程的源码都放于 src 文件夹下。
包名与文件夹可以任意安排，但是要注意，package 的是包名，import 里面的是路径名，import 成功后在代码中引用其它包中的函数时，形式是 包名.函数名

## 3.批处理
在批处理是若使用了 !n! ，则一定要加上： setlocal enabledelayedexpansion

bat 比较两个变量是否相等，对于整数来说，使用 equ ，字符串使用 ==，另外需要注意的是，需要这样的形式：
if "%value%"
即需要用双引号包围

十分注意， if 和 else 后面要有空格，因为有可能导致括号与变量或值被解析为一个整体，这会运行出错

if 体内不能是空块，如果是空，加一点注释即可。

()中无法写注释,因为bat把它()当成一行语句,这样注释就相当于一行中语句一部分。

## 4.Idea IntelJ 编译
idea --> Run  --> Configuraions -->JRE 这个是设置整个工程的 jre，而非所有 modules
在 Moudle 上右键 --> open Moudle settings --> Dependencies 设置 jdk
或者 File --> Project Structure --> Modules -->Dependencies 设置 jdk

## 5. http 代理
对于不能设置网络代理的程序，可以使用 charles 在本地开启一个代理服务器(如果有需要就设置 external proxy)。最后确认 charles 是否已经设置了 windows proxy。所有的事情就搞定了，任何程序都会把请求发送到 charles

## 6.学习看机器码
简单地看机器码
https://www.cnblogs.com/guocai/archive/2012/10/18/2730048.html

## 7.图像与机器学习
libsvm 解决的问题是，利用特征数据进行训练得到模型，之后进行预测
caffe 可以用来提供图处的特征； 如何理解  caffe，https://www.zhihu.com/question/27982282

图像学习之如何理解方向梯度直方图（Histogram Of Gradient）
https://www.leiphone.com/news/201708/ZKsGd2JRKr766wEd.html

汉字常用特征的提取方法详解
http://blog.csdn.net/jy02660221/article/details/52012543

计算图像一阶微分参见这里：http://blog.csdn.net/jia20003/article/details/7562092

特征提取; http://blog.csdn.net/qq_25543287/article/details/71296100

## 8.opencv2
下载的版本是 2.4.9 ，已经编译好的几个版本是：VC10, VC11, VC12，而自己只有 VC9，在此情况下，需要自己编译出 VC9 的版本。下载 CMake 工具，按教程重新编译。完了之后 Debug 和 Release 各编译出一个版本。
生成的 dll ，名字后面带 d 的是 debug 版本，不带 d 的是 release 版本。相应模式的应用程序在使用 opencv2 库时要注意选择正确。

## 9.imgSearch 研究日志
- 使用 kmeans 算法得到汉字特征
- 要把参加 kmeans 点的个数提上来，才有意义
- 根据汉字在图片中的规律，使用窗口扫描灰度化后的图片，若这个窗口内含有的像素点超过一定的数目则这个窗口中心可以成为一个中心
- 若参加聚类的像素点数目存在量级差别，则不是一张图？
- 搜索窗口中的点个数，优化为：二分查找
- 函数传递参数时，优化为使用指针传递
- 如果初始的质心有较大不同，则说明两个文字有较大可能不同
- 是否可以不识别文字，直接人眼来分类
- 如何提取汉字特征？
- 只识别其中一个字，就可以识别整个词语？
- 是否可以记录大图(不含文字)的特征，直接记录答案
- 当前解决：相同的 imgindex 不同照片(存在完全相同的大图)
- 可能需要对每个 clip 有多个数据采集区
- 为什么生成的图像的 width 和 height 反了: New3DSlice 的第一个参数是 height，第二个是 width
- 发现了两个 bug: 1. imgo 的计算灰度图算法，应该取 rgb 三色平均，而实际上它取的是 rga; 2. goleveldb  iterator 返回的 key 不是复制出来的，而是缓存地址，这可能导致误用。
- 判断字形作为特征：左右结构，上下结构，封装结构，等等，
- 验证了相同的大图出自于缓存，而非固定存储。 -- 可能缓存的周期是每天，则每天下载一次数据，验证大图的总数量。
- 挖掘小图的协同出现关系：同一主题下的小图伴随出现的概率应该远大于非同一主题。
- 存在一个 bug : 图片的 id 是使用字母+数字形式的，而 leveldb 默认的遍历顺序是字典序的：A99 > A200。这会导致遍历顺序出乎意料。
- leveldb OPenOPtion 中的 BlockSize 设置多少存储图片较合适
- 各个  iterator 有自己的缓存吗 ？如果共享一下，在多个迭代器分别访问时，存在竞争，且换入换出的代价
- leveldb 性能调优：设置布隆过滤器， 设置缓存大小
- goleveldb 一个坑：生成 iterator 之后，有时需要手动调用 iter.First ，后续 iter.Valid 才会成功
- 局部敏感 hash 或 全局 hash：小图非常非常相似，二进制数据也非常相近，目前的方法，采用顺序取样得到小图索引，结合 leveldb 数值型的索引存储，则取出时，两个相近的小图可能相隔的非常远（比如采样数据的第一个字节就相差较大）。这是一个可以优化的方向
- 待解决的问题本质上是：如何将相似的小图，邻近存放
- golang 中  map 类型的键若是复合类型，怎么办
- 上面的疑问的答案：由于 golang 无继承及强制实现接口，故无法限制结构体必须有某个方法(hash 和 equals)，所以无法将结构作为 map 的 key
- 问题： golang 返回函数内部变量的地址，函数调用结束后，是否可以访问那个变量
- 待做：clip index 算法优化：1.分多段建立多个索引 2.局部敏感 hash 3.容错 hash: 像素点数值阶段化。 clip 协同关系分析
- clip 图片使用图像转换软件转换试试
- 已经可以输出 dbId, mainImgId, which 对应的 clip 的 index ，可适当分析
- 优化 imgId, clipId 编码
- 目前皆是明文编码： imgId 是 8 个字节存放
- 多个 iterator 同时对 leveldb ，可能导致读取的效率低：缓存不停的换入换出，所以要“集中读”，
- DB::NewIterator 实现： DB::NewIterator 每次调用会生成以下迭代器：
1.mem_->NewIterator
2.imm_->NewIterator
3.每个第 0 层文件构成一个迭代器
4.从第1层开始，每层文件构成一个迭代器
mem_ 和 imm_ 皆是内存中的 skiplist，整个数据库只有这两个，生成 iterator 并不会有复制器出来。
每个第 0 层的文件构成的迭代器，均放在 Version 对象的 table_cache_ 中：若某个 sst 文件不在内存中，则读取 sst 文件的 footer 和 index_block 到内存(注意，此时并未读取 data_block )。后续在查找时，若某个 Block 不在内存中，则加到全局的 block 缓存器中：options 中定义的：Cache* block_cache;，默认的大小是 8<<20 即 8 M。
从第 1 层开始的每层一个迭代器，也是依托在 table_cache_ 的：每层是一个 TwoLevelIterator，先通过第一个迭代器， 定位 user_key 到某层某文件，再通过第二个迭代器，定位某层某文件所在的位置(如缓存的位置)，最后，在那个位置查找 user_key 的值。
整个数据库只有一个  Version，从而也只有一个 table_cache。
table_cache 的组织方式是：
1.使用一个 LRUCahce 对象控制和缓存打开的文件信息(table_cache.cc 中 ,默认最多只能 1000-10，即 990 个)
2.使用一个 LRUCache 缓存了所有打开的 Block(table.cc 中 BlockReader 函数)
LRUCache 的行为是，淘汰最近最久未使用的缓存。
由以上可知，当有多个线程同时进行顺序访问时，由于 block_cache 较小，全局唯一的  block_cache 会经常被换入换出，导致性能降低。
若每个 block 块大小为 2M，当有 N 个线程时，比较适当的一种设置方式是，block_cache 大小应该为至少 N*2 兆比较合适：这可以避免某个线程加载了 2M 数据到缓存，不会立即被换出。
- leveldb 迭代器依序访问效率低，会不会是命中了太多次 compaction，改日要研究一下 compaction 的触发规则
- idea 编辑器在运行 go 程序时，貌似会抢占到 go 编译器，导致在编辑器外面使用 go build 编译 go 文件陷入等待。
- 扩增 img_key 时要注意一点，同一个线程所属图片，一定要存储在相领的块。否则会导致后面的读取缓存被换入换出
- 同时需要注意，多个线程同时写数据库，可能使得相同 threadId 的数据不在同一个块。可能是离散的。
- golang 坑:
golang 坑：笔误导致非常难追踪的错
全局定义 var threadFinished chan int
在创建线程处，定义： threadFinished:=make(chan int, 4); 此时使用的并非那个全局变量，而是局部的。
尤其是此全局变量被两波使用，各波都创建了一堆线程，这将导致很难追踪的问题。
另外，golang 并发执行时一定不要忘记了 go 关键字
- golang 顺序地使用 Iterator.Next 可能速度较慢，即使逻辑上的有序性与物理存储有序性一致，但是由于 Iterator 结构的复杂性，在并行场景下要注意缓存换入换出的问题，而且 Next 的实现内部包装了多个迭代器。可能这是较慢的原因。
实际上，最大的一个导致遍历速度慢的原因是，触发了 comparation。
如果知道存储内容的有序性，可以直接使用 Seek ，自己计算出每一次遍历的 key，这样可以提速
- []byte 转换为 string 时，会有内存的拷贝：使用 str := (*string)(unsafe.Pointer(&bytes)) 则可以避免内存拷贝
- leveldb 写入数据完成后，造成记得要手动 CloseDB，否则在写入缓存区中的数据可能不会被 Flush 到磁盘。
- 估计还有一种可能会大幅度降低写入速度：当写入的键与已存在的键重复度较大时，一是会导致有大量的无用记录(leveldb 并不会立即对提交的删除作存储层面的删除，而是会插入一条相同 user_key 的记录，但 seqencenumber 更大，即更新，这会导致后续的访问会得到“删除”：无值。逻辑上是正确的，但是这些存在的无用的记录会降低遍历速度。)，这些相同 user_key 的旧版本记录被删除只可能发生在 compact 且它们必须要同时处于同一个 compact 单元：这是非常难以满足的。
另一方面，由于 leveldb 快照功能的存在，若使用了快照这个功能，会使得同一 user_key 被保留很多份，不能被删除。
综上，为了提高“高频率碰撞”写入时的速度，需要使用 leveldb.Batch，将邻近的写入打包到同一单元，然后再一起写入。可以避免并发环境下各线程连续写入，相互打散写入，导致相同 user_key 的多个版本分布在不同的 sst 文件，甚至不同的 level 中，这便降低了它们合并的概率。
另外，前面有提到，相领的键之记录，最好要保存到同一个块，这可以大幅度提高顺序访问效率。
- clipmain 过程可能有错误，求原value时，应该参考在缓存中的值。
- download 是 imgid 分散的来源，可以考虑使用缓存。各个线程分别批量写入。++++++然而这样也有问题，leveldb底层对于并发的批量写，可能也会被打散，需要更多考虑
- 可否拆开为多个 clip，分而治之-------可能没有意义：处理一个库时 cpu 占用已经接近100%了
- 打印clip索引长度，是否已经归十化？是否相邻的 clip index 对应的照片一致呢？
- edit hash 改进：hash 前面几个字节的容错性应该更加好：前几个字节分支hash
- map本质上是一个字典指针
- 非常重要的一点是， golang 弱化了程序员需要关注栈与堆的行为。分配在堆与栈上，完全由 golang 编译器决定。但它们都指示了一个正确的行为：基于对象的引用法则：需要保证引用有效。当一个对象只在函数内被使用，则可以在栈上分配，否则，函数调用完毕后仍然需要使用此对象，则只能在堆上分配。
- 但 golang 比较自作聪明的地方是，若一个对象逃逸出了函数作用域，但逃不远，有可能还是会在栈上分配：在调用者的栈上分配即可。
- 为一个没有成员的结构体生成对象时，它们都指向同样的一个对象。这一点通过打印它们的地址得知：0x55b988 这是一个较“短”的地址，可能是常量区？正常的对象的地址类似：0xc04204c0c0。
当结构体中加入一个变量时，则立即体现不同了。
- c++ 中的 this，暗含的指向当前对象的指针，不可以被增值。这是显然的。而在 golang 中，没有 this，不受此限制：

以上说明， 指针作为值传递，函数作用域内的修改，对之后的外围作用域没有影响。





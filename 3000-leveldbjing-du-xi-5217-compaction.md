#leveldb 的 Compaction
##概述
本文讲述 leveldb 的 compaction，包括为什么要 Compation，策略(size 模式和 seek 次数模式)及原因，怎么 Compaction。
Compaction 意即“压”，在 leveldb 中表示将内存中的 table 导入到磁盘和将低层次 sst 压入到高层次。

##为何需要 Compaction
- 1.我们为了查询效率，需要将每一层所有的 sst 文件保持有序性，再通过记录这些 sst 文件的键值范围（即 sst 索引）。当查询键对应的值时，就可以在索引中使用二分查找，迅速定位到目标键所在的 sst 文件，
然后再展开查找。
- 2.正是因为这种有序性，在接收并插入一个数据时，需要将它插入到合适的 sst 文件中，当目标 sst 文件已经足够
大时，需要重新整理 sst 文件：拆分，重新组合这些 sst 文件。
- 3.更进一步地考虑，有一些“冷数据”，它们写入后很少被访问，它们不应该放在“热查找”位置，显然这些“热查找”位置应该是数据量少，
查找速度快的位置。这也是为什么 leveldb 规定，level+1 层的文件数量是 level 层的 10 倍(至于为什么是 10 倍的解释参见 sst 结构)，即越低的层次越是“热查找”位置。
- 4.基于程序的局部性思考，我们有理由认为新插入的数据有可能会被立即访问，因此，这些新数据应该被放在“热查找”位置-低层次的 sst 文件上。
- 5.低层次 sst 容量较小，我们需要将一些数据换出到较高层次中。这即是 compaction 的动机。

##Compaction 策略
leveldb 中 compaction 有两种策略，下面是较直观的解释：

size_compaction: 	基于文件大小触发的 compaction. 这样做的动机是 leveldb 的层次控制: 高层的文件数量是低层的 10 倍

seek_compaction:	基于文件“被经过但不命中”的次数达到阈值触发的 compaction. 
					这样做的动机是消灭稀疏的文件, 将它放入更高层的文件中, 提高文件的稠密度提高查询效率.
					文件太稀疏了: 文件的区间(smallest--largest)太大了, 含有的 kv 对只有那么多, 很多查找的键落在此范围但不被命中
					这会形成查找的阻碍，将此文件填充到高一层文件中, 提高查询效率

优先执行的是 size_compaction。

Version 管理详细见相关章节。	

##Compaction 的输入文件
###基本输入文件
根据 Compaction 策略，我们先初步求出需要对哪些文件进行 Compaction，它们是 Compaction 的输入文件。
- 1.size_compaction 策略
在 size_compaction 策略中，通过 compaction_level_ 指出应该在哪一层上进行 compact，而 compact_pointer_ 指出了在这层的哪个文件开始执行
compact：compact_pointer_ 指出了各层上一次 compact 的位置。此次 compact 的对象是这些位置后面的文件。若 compact_pointer_ 为空则从第0个文
件开始。
compaction_level_ 是在每次版本发生变化时（VersionEdit 被 Apply），动态计算出来的，依据是：
	a).第 0 层文件总大小达到阈值
	b).第 n 层文件总大小达到阈值某个倍数(即 score，当前 leveldb 设定为 1)
compact_pointer_ 正是在每次确认 compaction 输入文件后被更新。

- 2.seek_compaction 策略
这是一个更为智能但更复杂的策略。
在数据库 seek 过程中, 对于第一个被遍历到的文件，若只是经过它, 并没有在此文件中命中, 还要继续在其它文件中查找, 则此文件需要被记录下来, 
用于后续的 compact。
这样做的依据是: 在查找元素时，此文件以“被经过但不命中”的方式多次访问, 次数多达 allowed_seeks 次, 那么说明此块对于全局的查找构成阻碍已经 
到达了一个程度, 此文件需要被"消灭" -- compaction 到更高层，背后更深层次的原因是, 此文件太稀疏了: 文件的区间(smallest--largest)太大了, 
很多查找的键落在此范围但不被命中，这会造成查找的效率低下，那么此文件应该被整理，直接填充到上一 level 文件, 使上一 level 文件更稠密.
此策略中，file_to_compact_level_, file_to_compact 指示了哪一层和哪一个文件需要被 compact。这两个值会在 DBImpl::Get 中被修改。

最后，输入文件分为 c[0] 和 c[1]，表示第 level 层和 level+1 层的输入文件。

###扩展输入文件
由上一点中的输入文件，扩大范围，求得最终的输入文件
	1).若当前 Compaction 的 level 为0，则将与 c[0] 键值相交的第0层文件，加入到 c[0]。
	2).否则:
		2.1)由 c[0] 与 level+1 层相交的文件得到 c[1], c[0],c[1] 键值范围是 (all_start, all_limit)
		2.2)若 c[1] 不为空则继续扩展: level 层与 (c[0] 并 c[1]) 相交的文件集合得到 expanded0, 转到 2.3)；否则转到 2.5)
		2.3)若 expanded0 的大小大于 c[0] 则继续扩展: level+1 层与 expanded0 相交的文件集合得到的 expanded1；否则转到 2.5)
		2.4)若 expanded1 的大小大于 c[1] 则扩展至此, c[0]=expanded0, c[1]=expanded1, c[0] 范围是 (smallest, largest), c[0]并c[1] 
			范围是 	(all_start, all_limit)；否则转到 2.5)
		2.5)计算 c[0]并c[1] 与 level+2 相交文件集合作为 c->grandparents_，将 level 层下一次 compact 的起始位置设置为本次 level 层上
		compact 的上界  c[0].largest
	
	以上如行云流水般的扩展输入文件的做法，本质上是：
		a).c[0]与 level+1 层的文件求相交得到 c[1]：是为了防止 level 层的文件 c[0] compact 到 level+1 中造成重叠，这会
			导致逻辑错误，违背“除了第0层外所有层上的文件各不相交”，所以要将 c[1] 这些文件单独拿出来，与 c[0] 一起，compact 到 level+1 层
		b).因 a) 中已将 c[1] 单独拿出，所以 level 中与 c[1] 相交的文件可以 compact 到 level+1 层，这一部分 compact 到 level+1
		不会与没有 compact 的部分发生重叠，这一部分加上c[0]，记为 expanded0
		c).因 expanded0 将会 compact 到 level+1 中，所以需要找到它与 level+1 层的相交，同样它需要从 level+1 拿出，再 compact 回去，
		这一部分加上c[1]，记为 expanded1，
		d).若 expanded1 和 expanded0 较原来的 c[1], c[0] 都要大，则说明扩展是有效的，接下来便用 expanded0 和 expanded1 执行 compact。
	
					
##Snapshot
在 compaction 这一章顺带讲 snapshot 是合适的，因为它与 compaction 关联实在太紧密了，而它单独成一章又无从讲起了。
snapshot，即快照，赋予了使用者查找在过去某一个瞬间数据库状态的能力：在那一时刻，整个 leveldb 有哪些数据，什么版本。
leveldb 会冗余地存储很多旧版本的数据。我们有可能误删了数据， 或者想追踪数据的变化历史，这时快照功能就十分有用了。
实现快照功能有以下两个主要问题：
- 1.前面说过，应该冗余存储旧版本数据。那么应该保留多老的数据呢？
联想到 leveldb 中任何一个数据(键值对)都有一个整数版本号 sequence number，我们也可以用一个整数来告诉 leveldb 对每个 user_key 都应该至少保留一个小于这个整数的版本(如果这个 user_key 有小于此整数的版本的话)。这个整数即是 snapshot。
只要 user_key 在那个时刻(产生 snapshot 这个版本的时刻)有版本(即 user_key 存在小于等于 snapshot 的版本)， 则我们应该返回这个时刻当
时最新的版本，即 user_key 的所有版本中小于等于 snapshot 的最大版本。
snapshot 的值必然比当前 leveldb 最新版本号小：最新版本号时时刻刻都是整个 leveldb 中最大的版本号。
- 2.结合1，我们应该如何使用 snapshot 以对每个 user_key 保存一个小于等于它的版本呢？我们在每次新写入时不可能做到这点，最新版本号作为
snapshot 没有意义：违背了快照的初衷，快照是历史的某时刻状态，并非现在。除了新写入数据，后面唯一一个可以动数据的地方就是 compaction 了。
leveldb 通过在 compaction 时给出一个 snapshot，在执行 compaction 时，会为每个 user_key 保留一个小于等于 snapshot 的版本(如果有的话)，具体
做法详见 ***怎样 Compact***。
- 3.（犹豫良久还是决定加上这一点）实际上 compaction 时被指定的并非是单个的 snapshot，而是一个 snapshot 集合(链表)，这是由于 snapshot
可以动态调整。当指定了 snapshot 集合时，因为保留的逻辑是：“为每个 user_key 保留一个小于等于 snapshot 的版本”，所以只需要取出集合中最小的
snapshot，执行此逻辑保留下来的各个 user_key 的版本范围是最广的，可以为所有的 snapshot 实现快照。


那么如何使用快照呢？ -- 查询数据时，通过option传入snapshot参数，查找时会跳过版本号比 snapshot 大的键值对，定位到小于等于 snapshot 的最新版本， 从而读取那个时刻的历史数据。

		
##怎样 Compact
依据上面两种 compaction 策略，选择出待 compaction 的文件（其实还有一些优化，详见 PickCompaction）：level 层和 level+1 层，将这两个层次的 sst 文件包装为一个 MergingIterator，按序遍历键值对，一个一个判断，是 drop 掉，还是加入到新的 sst 文件中。
###drop
- 1.对于当前遍历到的 user_key，已经为它保留了一个小于等于指定 snapshot 的版本，当前遍历的版本可以 drop。
- 2.首次遍历到的 user_key，不论版本号与 snapshot 关系如何，都应该保留下来。注意，即使当前版本是一个 deletion，也应保留。理由如下：
a).这个 user_key 最新版本的值类型是 deleteType：则这个 user_key 在之后的遍历中都不应该再被查找到。但是如果删除了这个 deletion 版本，此
user_key 的旧版本会被遍历，而一旦这个旧版本是一个正常的 valueType，则它会被查找到返回给用户，这是逻辑错误。因此如果要删除这个 deletion
版本，则需要删除此 user_key 的所有版本：让这个 user_key 不可能被查到。基于这点，有以下两个分支判断：
a-1).这个 deletion 版本的版本号小于等于 snapshot ：删除所有的历史版本，相当于人为地扼杀了 user_key 本应该具有的快照功能。
a-2).这个 deletion 版本的版本号大于 snapshot，有以下两个分支判断：
	b-1)若后续的遍历中 user_key 存在小于等于 snapshot 的版本，则与上面 a-1) 一样，扼杀了快照功能。
	b-2)若后续的遍历中 user_key 不存在小于等于 snapshot 的版本，并没有扼杀对于 snapshot 的快照功能。弊端不很明显，在于，compaction 指定的
	是一个版本号数组，虽然没有扼杀最小的 snapshot 的快照功能，但是有可能扼杀较大的 snapshot 的快照功能。不扼杀的情况是：此 user_key 的最老
	的版本号大于最大的 snapshot。但是对于其它的 user_key，可能仍然会扼杀快照功能。完全搞清楚：是否对于所有的 user_key 没有扼杀任何一个 snapshot 的快照功能，是一件非常繁杂的事情。因此简单而且稳妥起见，不要 drop 这个 deletion 版本即可。

综上，结论是：首次遍历到的 user_key 不可以 drop。
- 3.非首次遍历到的 user_key 是一个 deletion 版本，则 drop 依赖于以下两个条件满足：
	1).当前 user_key 只出现在执行 compaction 的层次(设为 level) 和 level+1 层中
	2).当前遍历的版本号小于等于 snapshot

原因如下：
1.当前是要将 level_ 层和 level_+1 层 compact，虽然在这两层将 user_key 的这个 deletion 版本删除了，但如果当前 user_key 还出现在比 level+1 层更高的层中，这些稍老的版本（层次越高数据越老越“冷”）会响应对 user_key 的查找，这将是逻辑错误。
2.若当前这个 deletion 版本号大于 snapshot 仍然将它 drop 掉，为了保持逻辑的正确性，还需要将此 user_key 后续版本全部 drop（第2点中讲过）。但
这样也有问题：会扼杀快照功能（上面详细阐述了这点）

将不可 drop 的键值对加入到新 sst 文件并写入到磁盘，再将它们纳入到 Version 管理，删除一些不会再引用到的文件，即完成了 compaction。
Version 管理详细见相关章节。其它细节见后面代码注释

###加入新的 sst 文件
若将当前键值对加入到当前构建中的 sst 文件，导致它与 level+2 层文件有较多重叠，则应该放弃加入：
如果这个 sst 文件与 level+2 层有重叠程度较高(按 leveldb 预定义，和 level+2 达到多于 10 个文件重叠)，则不能将这个文件直接放入 level+1 层中，
因为如果后续这个 level+1 层文件发生 compact，则我们需要对 level+2 层多于 10 个文件进行 compact，这是很高的耗费。
因此，我们要停止这个 sst 构建并导出到磁盘，开始新一个 sst 的构建。以上即是 ShouldStopBefore。
实际上，ShouldStopBefore 有两个优化：
- 1.计算 compaction 输入文件时，也一并计算出输入文件与 level+2 层的哪些文件相交：grandparents_，在判断 sst 与 level+2 重叠程度时，用这个更
小的集合去判断可以提高效率
- 2.判断 sst 与 grandparents_ 重叠程度时，并非一起判断，而是对 compaction 流程中依序处理到的键值对，逐个判断的：利用有序性，使得判断的复杂度
是 O(n)，详见 ShouldStopBefore 的代码注释

##Compact 优化
compaction 中有一种场景可以优化，使 compact 的速度大大提高：当输入文件 c[0] 只有一个文件，c[1] 为空时。也即整个 compaction 只需要处理 level
层的一个文件。当这个文件与 level+2 层重叠的程度较小时，我们直接将这个文件 compact 到 level+1 层即可。这个优化称为“IsTrivialMove”。

因此，正确的做法是，将这个 level 层的文件在 compact 时，拆分为多个文件放入 level+1 中，使得它们每一个与 level+2 层键重合的文件数量保持
最多10个。


##DoCompactionWork

```
/************************************************************************/
/* 
	lzh: 压缩挑选出来的 level 层和 level+1 层文件集合: c[0], c[1]

	leveldb 中的 snapshot 的功能是，
	Get 时可以通过option传入snapshot参数，那么查找时会跳过SequenceNumber比snapshot大的键值对，定位到小于等于 snapshot 的最新版本，从而完成快照功能：读取历史数据。
	
	对应地，compaction 应该有如下两个逻辑：
		1.为每个 user_key 保留一个小于等于 snapshot 的最大版本号(如果有的话)。
		2.最基本的，各个 user_key 要保留其最新的版本，不论这个版本是不是小于 snapshot。
	
	由1，2两点，当 snapshot 没有被设置时，snapshot 应该被默认为整个 leveldb 中当前最大版本号。此时第1点等于无（此时它即是第2点）。
	另外，一个解析出错的键值对后面紧挨着的键值对不会被 drop。
	
*/
/************************************************************************/
Status DBImpl::DoCompactionWork(CompactionState* compact) {
  const uint64_t start_micros = env_->NowMicros();
  int64_t imm_micros = 0;  // Micros spent doing imm_ compactions

  Log(options_.info_log,  "Compacting %d@%d + %d@%d files",
      compact->compaction->num_input_files(0),
      compact->compaction->level(),
      compact->compaction->num_input_files(1),
      compact->compaction->level() + 1);

  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == NULL);
  assert(compact->outfile == NULL);

  //lzh: 函数注释已经阐明 snapshot 没有被设置时，应该被设置为 leveldb 中当前最大版本号。
  //lzh: 若有多个 snapshot 应该使用最小的那个，以保证更古老的版本数据能够被保存下来
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->number_;
  }

  // Release mutex while we're actually doing the compaction work
  mutex_.Unlock();

  //lzh: 得到遍历所有数据的 Iterator. 它将正序遍历所有的 Internalkey，相同的 user_key 越新的版本越前面被遍历到
  Iterator* input = versions_->MakeInputIterator(compact->compaction);

  //input = getTestIterator(compact->smallest_snapshot);

  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;	//当前正遍历到的 internal key

  std::string current_user_key;	//当前遍历到的 user_key。

  //lzh: current_user_key 是否有效。还没有开始遍历，或者解析失败时此值为 false
  bool has_current_user_key = false;

  //lzh: 上一次遍历到的当前 user_key 的 sequence number。
  //lzh: 还没有开始遍历，或者解析失败时，或者当前的 user_key 是首次遍历到，则此值为 kMaxSequenceNumber。保证每第一次遍历到的 user_key 不被 drop
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;

  //lzh: 依次遍历, 若不 drop, 则 compact
  for (; input->Valid() && !shutting_down_.Acquire_Load(); ) {
    // Prioritize immutable compaction work
    if (has_imm_.NoBarrier_Load() != NULL) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != NULL) {
        CompactMemTable();	//lzh: 将内存中的 mem 导出到磁盘上的 sst 文件(作为第 0 层)
        bg_cv_.SignalAll();  // Wakeup MakeRoomForWrite() if necessary
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    Slice key = input->key();
	//lzh: 是否应该停止往当前构建的 sst 中增加 key. 若返回 true 则需要将当前的构建写入到磁盘然后再重新开始一个 sst 的构建
	//lzh: 首次调用 ShouldStopBefore 必然返回 false. 
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != NULL) {
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }

    // Handle key/value, add to state, etc.
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else 
	{
	  // lzh: 首次遇到 ikey.user_key 这个 user_key
      if (!has_current_user_key ||	//lzh: !has_current_user_key 表示上一个 ikey 解析失败或者当前是第一次解析
          user_comparator()->Compare(ikey.user_key,
                                     Slice(current_user_key)) != 0) {	//lzh: 表示当前的 ikey.user_key 与 current_user_key 不相等
        // First occurrence of this user key
		
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;	//lzh: 当前 ikey.user_key 是首次遇到，设置 last_sequence_for_key=kMaxSequenceNumber 保证当前 ikey 不被 drop
      }
	  
	  //lzh: 当前 user_key 的上一个版本 <= snapshot，我们把上一个版本保留下来，以满足 snapshot 的功能：为每个 user_key 保留一个小于等于 snapshot 的最大版本号。
	  //lzh: 从当前版本开始(包括在内)，后续所有当前 user_key 的更旧版本可以 drop 了。
      if (last_sequence_for_key <= compact->smallest_snapshot) {
        // Hidden by an newer entry for same user key
        drop = true;    // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) 
	  {
		  //lzh: 若当前 user_key 上一个版本 > snapshot，但当前 user_key 的 type 是 deletion，那么什么情况下可以 drop 掉这个版本呢？
		  //lzh: 1.当前是要将 level_ 层和 level_+1 层 compact，如果当前 user_key 出现在比level_+1 层更高的层次，那么这个 kv 必然不能被 drop 掉，
		  //lzh: 否则，其它层可能存在的更旧的版本就会被使用，这将导致逻辑错误。
		  //lzh: 2.若当前版本大于数据库设置的最小快照号 smallest_snapshot 仍然将它 drop 掉
		  //lzh: 注意当前的 user_key 是 deletion 版本，为了保持逻辑的正确性：相同 user_key 的 deletion 版本后面的版本是无效的，还需要将此 user_key 后续版本全部 drop
		  //lzh: 但是这样一来快照功能的逻辑就错了：我们的查找范围是所有小于等于 snapshot 的版本。因为我们已经删除了所有的较旧版本的 user_key，所以使用快照功能也无法访问了，
		  //lzh: 虽然在一个较旧的时刻，user_key 最新的版本是有效的，在快照下应该是可访问的。
		  //lzh: 综上，所以判断的条件有以上三个。
		  
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }

	  //lzh: 在赋值之前 last_sequence_for_key 指的是前一次遍历到的当前 user_key 的版本。若当次是首次遍历到当前 user_key 则此值在赋值前为 kMaxSequenceNumber
      last_sequence_for_key = ikey.sequence;
    }
#if 0
    Log(options_.info_log,
        "  Compact: %s, seq %d, type: %d %d, drop: %d, is_base: %d, "
        "%d smallest_snapshot: %d",
        ikey.user_key.ToString().c_str(),
        (int)ikey.sequence, ikey.type, kTypeValue, drop,
        compact->compaction->IsBaseLevelForKey(ikey.user_key),
        (int)last_sequence_for_key, (int)compact->smallest_snapshot);
#endif

	//lzh: 当前 input->key 被 drop 掉, 不用 compact 到文件中
    if (!drop) {
      // Open output file if necessary
		//lzh: 如果首次进入此分支， compaction 没有初始化(没有生成 file_number, 没有生成文件, 建立 sst 对应的内存 table), 那么需要初始化
      if (compact->builder == NULL) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }

	  //lzh: 如果 sst 文件对应的内存 table 中没有数据, 则当前被加入的 key 作为 smallest. (注意 key 是正序遍历的, 所以此逻辑是正确的)
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
	  
	  //lzh: key 是正序遍历的，所以每次新加入的 key 就是最大值
      compact->current_output()->largest.DecodeFrom(key);

	  //lzh: (key, value) 加入到 sst 内存 table 中
      compact->builder->Add(key, input->value());

      // Close output file if it is big enough
	  //lzh: sst 内存 table 满了，需要 dump
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }

    input->Next();
  }

  if (status.ok() && shutting_down_.Acquire_Load()) {
    status = Status::IOError("Deleting DB during compaction");
  }

  //lzh: 最后一轮循环中正在构建的 sst 大小没有达到阈值还没来得及写入磁盘, 此处需要处理
  if (status.ok() && compact->builder != NULL) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  delete input;
  input = NULL;

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }

  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);

  //lzh: 最终将所有生成的 sst 文件应用到内存版本管理, sst 文件层次管理中
  if (status.ok()) {
    status = InstallCompactionResults(compact);
  }
  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log,
      "compacted to: %s", versions_->LevelSummary(&tmp));
  return status;
}


/************************************************************************/
/* 
	lzh:	
		要点1:
			leveldb 提供了两种触发 compaction 的条件/方式:
			size_compaction: 基于文件大小触发的 compaction. 这样做的动机是 leveldb 的层次控制: 高层的文件比低层的要大, 约 10 倍

			seek_compaction:	基于文件“被经过但不命中”的次数达到阈值触发的 compaction. 
								这样做的动机是消灭稀疏的文件, 将它放入更高层的文件中, 提高文件的稠密度提高查询效率.
								文件太稀疏了: 文件的区间(smallest--largest)太大了, 含有的 kv 对只有那么多, 很多查找的键落在此范围但不被命中
								这会形成查找的阻碍，将此文件填充到高一层文件中, 提高查询效率
		要点2:		
			c->inputs_[0] 是 level 层需要 compact 的文件集合
			c->inputs_[1] 是 level+1 层需要 compact 的文件集合
*/
/************************************************************************/
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  //lzh; 优先执行 size_compaction
  const bool size_compaction = (current_->compaction_score_ >= 1);

  //lzh: 在 db->Get 中若对多于一个的 sst 文件/缓存 seek 的次数过多, 则  current_->file_to_compact_ 会被设置
  const bool seek_compaction = (current_->file_to_compact_ != NULL);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level+1 < config::kNumLevels);
    c = new Compaction(level);

    // Pick the first file that comes after compact_pointer_[level]
	//lzh: 选出首个最大值大于 compact_pointer[level] 的 sst 文件
	//lzh: http://catkang.github.io/2017/02/03/leveldb-version.html Version中会记录每层上次Compaction结束后的最大Key值compact_pointer_，
	//lzh: 下一次触发自动Compaction会从这个Key开始。容量触发的优先级高于下面将要提到的Seek触发。
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||		//lzh: 本层 compact_pointer_ 以前的文件都已经被 compact 过
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0)	//lzh: compact_pointer_ 以后的(没 compact 过)文件纳入 compact 范围
	  {
        c->inputs_[0].push_back(f);
        break;
      }
    }

	//lzh: c->inputs_[0].empty() 说明 level 层所有的文件之前都已经 compact 过. 但是本函数仍被调用, 进一步说明由于某些情况(如本层 size 仍较大, 或触发了 seek_compaction)
	//lzh: 需要再次 compact, 所以从本层第一个文件开始.
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(level);

	//lzh: 将 level 层的 current_->file_to_compact_ 文件 compact 到 level+1 层
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return NULL;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // Files in level 0 may overlap each other, so pick up all overlapping ones
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    GetOverlappingInputs(0, smallest, largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }

  //lzh: 找到另外 compact 的输入文件
  SetupOtherInputs(c);

  return c;
}

/************************************************************************/
/* 
	lzh: 此函数被两处调用: PickCompaction 和 CompactRange

	c->inputs_[0] 简写为 c[0], c->inputs_[1] 简写为 c[1]
	
	1. PickCompaction 中 level 上的一个待 compact 的文件集合是 c[0]. (注意第 0 层的特殊情况. 详见 PickCompaction 最末的处理)

	2. 由上面的 c[0] 与 level+1 层相交的文件得到 c[1], c[0],c[1] 范围是 (all_start, all_limit)

	3. 若 c[1] 不为空则继续扩展: level 层与 (c[0] 并 c[1]) 相交的文件集合得到 expanded0, 转到 4. 否则转到 6.

	4. 若 expanded0 的大小大于 c[0] 则继续扩展: level+1 层与 expanded0 相交的文件集合得到的 expanded1. 否则转到 6.

	5. 若 expanded1 的大小大于 c[1] 则扩展至此, c[0]=expanded0, c[1]=expanded1, c[0] 范围是 (smallest, largest), c[0]并c[1] 范围是 (all_start, all_limit). 否则转到 6.

	6. 计算 c[0]并c[1] 与 level+2 相交文件集合作为 c->grandparents_. 计算 level 层下一次 compact 的起始位置是 largest, 即本次compact 的上界c[0].largest
*/
/************************************************************************/
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;
  GetRange(c->inputs_[0], &smallest, &largest);

  GetOverlappingInputs(level+1, smallest, largest, &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);

  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    GetOverlappingInputs(level, all_start, all_limit, &expanded0);
    if (expanded0.size() > c->inputs_[0].size()) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      GetOverlappingInputs(level+1, new_start, new_limit, &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d to %d+%d\n",
            level,
            int(c->inputs_[0].size()),
            int(c->inputs_[1].size()),
            int(expanded0.size()),
            int(expanded1.size()));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }

  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  if (level + 2 < config::kNumLevels) {
    GetOverlappingInputs(level + 2, all_start, all_limit, &c->grandparents_);
  }

  if (false) {
    Log(options_->info_log, "Compacting %d '%s' .. '%s'",
        level,
        EscapeString(smallest.Encode()).c_str(),
        EscapeString(largest.Encode()).c_str());
  }

  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}

```
# 术语

Iterator 迭代器: 用户使用迭代器按排序顺序查找范围内的键。参见: https://github.com/facebook/rocksdb/wiki/Basic-Operations#iteration

Point lookup 点查找: 在RocksDB中，点查找意味着使用Get()读取一个键。

Range lookup 范围查找: 范围查找意味着使用迭代器读取键的范围。

SST File (Data file / SST table) SST文件(数据文件/ SST表): SST表示已排序的序列表。它们是存储数据的持久文件。在文件中，键通常按照排序的顺序组织，这样就可以通过二进制搜索确定键或迭代位置。

Index 索引: 一个SST文件中数据块上的索引。它作为索引块保存在SST文件中。默认索引格式是二进制搜索索引。

Partitioned Index 分区索引: 二叉搜索索引块划分为多个较小的块。参见 https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters

LSM-tree LSM树: RocksDB 是基于 LSM-Tree的存储引擎。参见  https://en.wikipedia.org/wiki/Log-structured_merge-tree 

Write-Ahead-Log (WAL) or log 预写日志: 用于在DB恢复期间恢复尚未刷新到SST文件的数据的日志文件。参见 https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format

memtable / write buffer 内存表 / 写缓存: 存储数据库最新更新的内存数据结构。通常它是按排序的顺序组织的。并包含一个二进制可搜索索引。参见 https://github.com/facebook/rocksdb/wiki/Basic-Operations#memtable-and-table-factories

memtable switch memtable切换: 在此过程中，当前活动的memtable(当前写入到的那个)被关闭，变成不可变的memtable。同时，我们将关闭当前的WAL文件并启动一个新的。

immutable memtable 不可变memtable: 一个等待刷新的封闭memtable。 

sequence number (SeqNum / Seqno) 序列号: 每次写入数据库都会分配一个自动递增的ID号。这个数字与WAL文件、memtable和SST文件中的键值对相关联。序列号用于实现快照读取。压缩中的垃圾回收、事务中的MVCC以及其他一些用途。

recovery 恢复: 数据库失败或关闭后重新启动的过程。

flush 刷新: 将memtable表中的数据写入SST文件的后台作业。

compaction 压缩: 将一些SST文件合并到其他SST文件中的后台作业。LevelDB的压缩还包括刷新。在RocksDB中，我们进一步区分了两者。参见 https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#multi-threaded-compactions

Leveled Compaction or Level-Based Compaction Style 级别压缩 或 基于基本压缩的样式: RocksDB的默认压缩样式。

Universal Compaction Style 通用压缩样式: 一种替代的压缩算法。参见 https://github.com/facebook/rocksdb/wiki/Universal-Compaction

Comparator 比较器: 一个插件类，它可以定义键的顺序。参见 https://github.com/facebook/rocksdb/blob/master/include/rocksdb/comparator.h
 
Column Family 列族: 列族是一个DB中单独的键空间。尽管这个名称有误导性，但它与其他存储系统中的"列族"概念没有任何关系。RocksDB甚至没有列的概念。参见 https://github.com/facebook/rocksdb/wiki/Column-Families

snapshot 快照: 快照是运行中的数据库中逻辑上一致的时间点视图。 参见 https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#gets-iterators-and-snapshots

checkpoint 检查点: 检查点是文件系统中另一个目录中数据库的物理镜像。参见 https://github.com/facebook/rocksdb/wiki/Checkpoints

backup 备份: RocksDB有一个备份工具，可以帮助用户将数据库状态备份到不同的位置。比如HDFS。参见 https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F

Version 版本: RocksDB的内部概念。一个版本由一个时间点上的所有活动的SST文件组成。刷新或压缩完成后，将创建一个新的"版本"，因为活动SST文件列表已经更改。旧的"版本"可以继续被正在进行的读请求或压缩作业使用。旧版本最终会被垃圾回收。

Super Version 超级版本: RocksDB的内部概念。一个超级版本由SST文件列表("版本")和活动memtable列表组成。压缩或刷新，或memtable开关都会创建一个新的"超级版本"。旧的"超级版本"可以继续被正在进行读取请求使用。旧的超级版本最终将不再需要后被垃圾回收。

block cache 块缓存: 内存中的数据结构，用于缓存来自SST文件的热数据块。参见 https://github.com/facebook/rocksdb/wiki/Block-Cache

Statistics 统计: 内存中的数据结构，包含活动数据块的累积统计信息。参见 https://github.com/facebook/rocksdb/wiki/Statistics

perf context perf上下文: 内存中的数据结构，用于测量线程本地统计信息。它通常用于测量每个查询的状态。参见 https://github.com/facebook/rocksdb/wiki/Perf-Context-and-IO-Stats-Context

DB properties 库属性: 函数 DB::GetProperty()可以返回一些运行状态。

Table Properties 表属性: 存储在每个SST文件中的元数据。它包括由RocksDB生成的系统属性和由用户定义回调计算的用户定义表的属性。参见 https://github.com/facebook/rocksdb/blob/master/include/rocksdb/table_properties.h

write stall 写停滞: 当刷新或压缩被积压时，RocksDB可能会主动降低写速度，以确保刷新和压缩可以赶上。参见 https://github.com/facebook/rocksdb/wiki/Write-Stalls

bloom filter 布隆过滤器: 参见 https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter

prefix bloom filter 前缀布隆过滤器: 一种特殊的过滤器，可以在迭代器中有限地使用。如果SST文件或memtable不包含前缀提取器提取的查找键的前缀，则可以避免一些文件读取。参见 https://github.com/facebook/rocksdb/wiki/Prefix-Seek-API-Changes

prefix extractor 前缀提取器: 可以提取键的前缀部分的回调类。这是前缀布隆过滤器中最常用的前缀。参见 https://github.com/facebook/rocksdb/blob/master/include/rocksdb/slice_transform.h
 
block-based bloom filter or full bloom filter 基于块的bloom过滤器或full bloom过滤器: 在SST文件中存储布隆过滤器的两种不同方法。参见 https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter#new-bloom-filter-format

Partitioned Filters 分区过滤器: 将一个完整的布隆过滤器划分为多个较小的块。参见 https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters 

compaction filter 压缩过滤器: 可以在压缩期间修改或删除现有键的用户插件。参见 https://github.com/facebook/rocksdb/blob/master/include/rocksdb/compaction_filter.h

merge operator 合并操作: RocksDB支持一个特殊的操作符merge()，他是对现有值的增量记录merge操作数。Merge操作符是一个用户定义的回调类，它可以合并 合并操作数。 参见 https://github.com/facebook/rocksdb/wiki/Merge-Operator-Implementation

Block-Based Table 基于块的表: 默认的SST文件格式。参见 https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format

Block 块: SST文件的数据块。在SST文件中。块以压缩形式存储。

PlainTable : SST文件的另一种格式，针对ramfs进行了优化。参见 https://github.com/facebook/rocksdb/wiki/PlainTable-Format

forward iterator/tailing iterator 向前迭代器/尾随迭代器: 一个特殊的迭代器选项，为非常特定的用例进行优化。参见 https://github.com/facebook/rocksdb/wiki/Tailing-Iterator

single delete 简单删除: 一个特殊的删除操作，它只是用户从未更新现有键时才有效。参见 https://github.com/facebook/rocksdb/wiki/Single-Delete

rate limiter 速率限制器: 它用于限制通过刷新和压缩写入文件系统的字节数。参见 https://github.com/facebook/rocksdb/wiki/Rate-Limiter

Pessimistic Transactions 悲观事务: 使用锁在多个并发事务之间提供隔离。默认的写策略被写入。

2PC (Two-phase commit) 悲观事务可以分两个阶段提交: 首先准备，然后实际提交。参见 https://github.com/facebook/rocksdb/wiki/Two-Phase-Commit-Implementation

WriteCommitted 写提交: 悲观事务中默认写策略，它缓冲内存中的写,并在事务提交时将它们写入数据库。

WritePrepared 写预编译: 悲观事务中的写策略，它缓冲内存中的写，如果是2PC事务或提交，则在准备时将它们写入数据库。参见 https://github.com/facebook/rocksdb/wiki/WritePrepared-Transactions

WriteUnprepared 写未预编辑: 悲观事务中的写策略，通过在事务发送数据时将数据写入数据库，从而避免需要更大的内存缓冲区。参见 https://github.com/facebook/rocksdb/wiki/WritePrepared-Transactions

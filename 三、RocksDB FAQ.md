# RocksDB FAQ

Q: 如果我的进程崩溃，它会破坏数据库吗？
A: 不会，但是如果禁用WAL日志，未刷新的mem-tables中的数据可能会丢失。

Q: 如果我的机器崩溃并重启，RocksDB会保存数据吗？
A: 当发出sync write(使用WriteOptions.sync=true) 调用 DB::SyncWAL() 或 memtables被刷新时，数据将被同步。

Q: RocksDB 会抛出异常吗？
A: 不会，RocksDB 返回RocksDB::Status来表明任何错误。但是，RocksDB 不会捕获由STL或其他依赖项引发的异常。
   例如，当内存分配失效时，您可能会看到std:bad_malloc，或者在其他情况下出现类似的异常。

Q: 如何知道存储在RocksDB数据库中的键数？
A: 使用GetIntProperty(cf_handle, "rocksdb.estimate-num-keys")来获得存储在列族中的估计键数，
   或者使用GetAggregatedIntProperty("rocksdb.estimate-num-keys",&num_keys)来获得存储在整个RocksDB数据库中的大约键数。

Q: 为什么GetIntProperty只能返回一个RocksDB数据库中大约键数？
A: 在任何像RocksDB这样的LSM数据库中获得准确的key数量都是一个具有挑战性的问题，因为它们有重复的key和删除条目。（即这将需要一个完整的compaction，以获得准确的键数）
   此外，如果RocksDB数据库包含合并操作符，那么估计的键数也会变得不那么精准。

Q: 基本操作 Put()、Write()、Get()和NewIterator() 线程安全吗？
A: 是的

Q: 我可以使用多个进程写入RocksDB吗？
A: 不可以，但是，它可以从多个进程以readonly只读模式打开。

Q: RocksDB支持多进程读访问吗？
A: RocksDB支持多进程只读进程，不需要写数据库。这可以通过调用 DB::OpenForReadOnly()打开数据库来实现。

Q: 当另一个线程发出读、写或手动compaction请求时，关闭RocksDB是否安全？
A: 不安全。RocksDB的用户需要在关闭RocksDB之前确保所有函数都已经完成。 

Q: 支持的最大键key和值value大小是多少？
A: 一般来说，RocksDB不是为大键设计的。键和值的最大推荐大小分别是8MB和3GB。

Q: 我可以运行RocksDB并将数据存储在HDFS上吗？
A: 是的，通过使用NewHdfsEnv()返回的Env， RocksDB将在HDFS上存储数据。但是，HDFS Env目前不支持文件锁。

Q: 我可以保存一个RocksDB的快照,然后将DB状态回滚它吗？
A: 可以，通过备份引擎(https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F) 或 检查点 (https://github.com/facebook/rocksdb/wiki/Checkpoints)

Q: 将数据加载到RocksDB最快的方式是什么？
A: 将数据直接插入数据库的快速方法:
   使用单写线程并按排序顺序插入
   将数百个key批处理为一个写批处理
   使用向(矢)量memtable
   确保选项: max_background_flushes至少为4
   在插入数据之前，禁用自动compaction，设置选项level0_file_num_compaction_trigger，选项level0_slowdown_writes_trigger 和 选项level0_stop_writes_trigger 设置为非常大。插入所有数据后，发出手动压缩。
   如果您将Options::PrepareForBulkLoad() 调用到您的选项，3-5将自动执行。

   如果您可以在插入之前离线预处理数据。有一种更快的方法：您可以对数据进行排序，并形生成范围不可重叠的SST文件，并隔离SST文件，
   参见：https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files
   
Q: RocksJava是否支持所有特性?
A: 我们正在努力使RocksJava特性兼容。但是，如果您发现缺少某些内容，则非常欢迎您提交pull request

Q: 谁在使用 RocksDB?
A: https://github.com/facebook/rocksdb/blob/master/USERS.md

Q: 删除数据库的正确方法是什么？我可以简单地在活动数据库上调用DestoryDB()吗？
A: 关闭数据库然后销毁数据库是正确的方法。在活动数据库上调用DestoryDB()是一种未定义的行为。

Q: DestoryDB() 和直接手动删除DB目录有什么区别？
A: 主要区别在于DestoryDB()将处理RocksDB数据库存储在多个目录中的情况。
   例如,可以通过为DBOptions::db_paths、DBOptions:db_log_dir 和 DBOptions::wal_dir指定不同的路径，
   将单个DB配置为将其数据存储在多个目录中。

Q: BackupableDB是否创建数据库的时间点快照？
A: 当BackupableDB::backup_log_files = true 或 flush_before_backup在调用CreateNewBackup()时被设置为true时，答案是肯定的。 

Q: 备份过程是否同时影响对数据库的访问？
A: 否，你可以同时读写数据库。

Q: 有没有更好的方法将Map-Reduce作业生成的键值对转储到RocksDB中？
A: 更好的方法是使用SstFileWriter，它允许您直接创建RocksDB SST文件并将他们添加到一个RocksDB数据库中。
   但是，如果要将SST文件添加到现有的RocksDB数据库中，则其键范围不能与数据库重叠。（https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files）

Q: 为不同的列族配置不同的前缀提取器是否安全?
A: 是，安全。

Q: 我可以更改前缀提取器吗？
A: 不可以，一旦指定了前缀提取器，就不能更改它，但是，可以通过指定null禁用它。

Q: 构建RocksDB锁需要的gcc的最小版本是什么？
A: 4.7，但是建议4.8或以上版本。

Q: 如何配置rocksdb来使用多个磁盘？
A: 您可以在多个银盘上创建一个文件系统(ext3、xfs等)，然后您可以在该文件系统上运行rocksdb，使用磁盘时的一些提示:
   
   如果使用RAID，那么不要使用大小的RAID条带大小(64kb太小，1MB就很好)
   考虑通过将ColumnFamilyOptions::compaction_readahead_size指定为至少2MB来启用压缩readahead。
   如果工作负载是写密集型的，那么就有足够的压缩线程使磁盘保持忙碌考虑启用async write behind进行压缩。

Q: 直接复制一个打开的RocksDB实例安全吗？
A: 不安全，除非以read-only只读模式打开RocksDB安全。

Q: 我可以用不同的压缩compression类型打开RocksDB，仍然读取旧数据吗？
A: 是的。因为rocksdb将压缩信息存储在每个SST文件中，并相应地执行解压缩，所以您可以更改压缩，db仍然能够读取现有的文件。
   此外，还可以通过指定ColumnFamilyOptions::bottommost_compression为最后一层指定不同的压缩类型。

Q: 如何配置RocksDB备份到HDFS中？
A: 使用BackupableDB并将backup_env设置为NewHdfsEnv()的返回值。

Q: 在压缩(Compaction)过滤器回调函数中从RocksDB读写是否安全?
A: 读是安全的，但在压缩过滤器回调函数中写入RocksDB并不总是安全的，因为当触发写停止条件时，写可能会触发死锁。

Q: RocksDB是否为快照保存SST文件和memtables？
A: 不是。有关快照如何工作，请参见（https://github.com/facebook/rocksdb/wiki/RocksDB-Basics# gets-iterators-andsnapshot）

Q: 使用DBWithTTL，是否有要删除过期键的时间限制？
A: 没有，DBWithTTL没有提供一个时间上限。过期的键在任何compaction过程中都将被删除。但是，不能保证什么时候开始这样的compaction，
   例如，如果您有一个从未更新的键范围，那么compaction就不太可能应用于该键范围。

Q: 如果我删除一个列族，而我还没有删除列族句柄，我还可以用它访问数据吗？
A: 可以，DropColumnFamily()只将指定的列族标记为已删除，直到它的引用计数为零并标记为已删除，它才会被删除。

Q: 当我只发出写请求，为什么RocksDB会从磁盘发出读取？
A: 这样的IO读取来自compaction，RocksDB压缩合并从一个或多个SST文件读取数据，执行类似合并排序的操作，生成新的SST文件，并删除它输入的旧SST文件。

Q: RocksDB 支持复制吗？
A: 不支持，RocksDB不直接支持复制。不过，他提供了一些API，可以用作支持复制的构建块。
   例如，GetUpdatesSince()允许开发人员迭代自特定时间点以来的所有更新。（https://github.com/facebook/rocksdb/blob/4.4.fb/include/rocksdb/db.h#L676-L687）

Q: RocksDB的最新稳定版本是什么？
A: 所有的稳定版本。https://github.com/facebook/rocksdb/releases。对于RocksJava，可以在https://oss.sonatype.org/#nexus-search;quick~rocksdb中找到稳定的版本。

Q: block_size表示是在压缩合并之前还是之后？
A: block_size表示在压缩合并前的大小。

Q: 使用 options.prefix_extractor前缀提取器，我有时会看到错误的结果。发生什么了?
A: options.prefix_extractor 前缀提取器有限制，如果使用前缀迭代，则不支持Prev()或SeekToLast(),而且许多操作也不支持SeekToFirst()。
   通过调用seek()和Prev()来查找前缀的最后一个键是一个常见的错误。但是，这并不受支持。目前没有办法找到前缀的最后一个键与前缀的迭代。
   此外，在完成要查找的前缀之后，不能继续迭代键。在需要这些操作的地方，可以尝试设置ReadOptions。total_order_seek = true禁用前缀迭代。

Q: 迭代器持有多少资源，何时释放这些资源？
A: 迭代器在内存中同时保存数据块和memtables。每个迭代器持有的资源是：迭代器当前指向的数据块。看到https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB#blocks-pinned-by-iterators
   创建迭代器时存在的memtables，即使memtables已被刷新。
   创建迭代器时存在于磁盘上的所有SST文件，即使它们被压缩。
   
Q: 我可以把日志文件和sst文件放在不同的目录吗？信息日志呢？
A: 可以。通过指定DBOptions::wal_dir可以将WAL文件存放在单独的目录中，
   也可以使用DBOptions::db_log_dir将信息日志写入单独的目录中。

Q: SST文件的bloom过滤器块总是加载到内存中，还是可以从磁盘加载？
A: 行为是可配置的。当BlockBaseTableOptions::cache_index_and_filter_blocks被设置为true时，只有在发出相关Get()请求时，才会将bloom过滤器和索引块加载到LRU缓存中。
   在另一种情况下，cache_index_and_filter_blocks被设置为false，那么RocksDB将尝试将索引块和布隆过滤器保存在内存中，直到DBOptions::max_open_files中有多少SST文件。

Q: 如果WriteOptions.sync=true 发出Put()或Write()操作, 这是否意味着之前的所有写操作都是持久性的?
A: 是的。但是只适用于以前所有带有WriteOptions.disableWAL=false的写操作。
 
Q: 我禁用了WAL，并依赖DB::Flush()来保存数据。这对单列族和有效。如果有多个列族，也可以这样做吗？
A: 不可以。当前DB::Flush()不是跨多个列族的原子性。我们确实有一个计划在未来支持它。
 
Q: 如果我使用非默认比较器或合并操作，我还可以使用ldb工具吗？
A: 在这种情况下，您不能使用常规的ldb工具。但是，可以使用这个函数rocksdb::LDBTool::Run(argc、argv、options)传递自己的选项，并编译它，从而构建定制的ldb工具。
 
Q: RocksDB如何处理读或写I/O错误？
A: 如果I/O错误发生在前台操作时中，比如Get()和Write(),那么RocksDB将返回RocksDB::IOError状态。如果错误发生在后台线程 options.paranoid_checks=true。我们将切换到只读模式，所有写操作都将被拒绝，状态代码表示后台错误。 
 
Q: 我可以取消特定的compaction压缩合并吗？
A: 不可以。您不能取消一个特定的compaction压缩合并。
 
Q: 当手动压缩正在进行时，我可以关闭数据库吗？
A: 不。那样做不安全。但是，您可以在另一个线程中调用cancelallbackfoundation(db, true)来中止正在运行的压缩，以便能够更快地关闭数据库。
 
Q: 如果我用不同的压缩样式打开RocksDB会发生什么？
A: 当使用不同的压缩样式或压缩设置打开rocksdb数据库时，会出现以下场景之一：
   1、如果新配置与当前LSM布局不兼容，数据库将拒绝打开。
   2、如果新配置与当前LSM布局兼容，那么rocksdb将继续打开数据库。然后，为了使新选项完全生效，可以需要完全压缩。
 
Q: 删除一系列键的最佳方法是什么?
A: 参见 https://github.com/facebook/rocksdb/wiki/Delete-A-Range-Of-Keys 
 
Q: 列族用于什么？
A: 使用列族最常见的原因:
   (1)在数据的不同部分使用不同的压缩设置、比较器、压缩类型、合并操作符或压缩过滤器;
   (2)删除列族以删除其数据;
   (3)一个列族存储元数据，另一个列族存储数据。
 
Q: 在多个列族和多个rocksdb数据库中存储数据的区别是什么？
A: 主要区别是备份、原子写和写的性能。
   使用多个数据库的优点：
   (1) 数据库是备份或检查点的单元。
   (2) 将数据库复制到另一个主机比将列族复制到另一个主机更容易。
   使用多个列族的优点：
   (1) 批量写是在一个数据库上跨多个列族的原子性的。使用多个RocksDB数据库无法实现这一点。
   (2) 如果向WAL发出同步写操作，太多的数据库可能会影响性能。
 
Q: RocksDB有列吗？如果没有列，为什么会有列族？
A: 没有。RocksDB没有列。有关什么是列族，请参见 https://github.com/facebook/rocksdb/wiki/Column-Families 
 
Q: RocksDB在读取中真的是"无锁"的吗？
A: 在下列情况下，读操作可能会持有互斥锁:
   (1) 访问分片块缓存。
   (2) 访问表缓存 如果 max_open_files != -1 设置。
   (3) 如果读取发生在刷新或压缩完成之后，它可能会短暂地保存全局互斥对象，以获取LSM树的最新元数据。
   (4) RocksDB所依赖的内存分配器(例如jemalloc)有时可能持有锁，这些锁很少持有，或者粒度很小。

Q: 如果我更新多个键，我应该发出多个Put()，还是将它们放在一个写批处理中并发出write()操作。
A: 使用写批处理对更多的键进行批处理通常比单个Put()执行得更好。
 
Q: 迭代所有键的最佳实践是什么？
A: 如果是小型或只读数据库，只需创建一个迭代器所有键。否则，请考虑每隔一段时间重新创建迭代器，因为迭代器将保存所有未释放的资源。
   如果需要从一致视图中读取，请创建快照并使用它进行迭代。
 
Q: 我有不同的键空间。我应该用前缀分隔它们，还是使用不同的列族？
A: 如果每个键空间都相当大，最好将它们放在不同的列族中。
   如果它很小，那么您应该考虑将多个键空间打包到一个列族中，以免维护太多列族的麻烦。

Q: 如果我触发一个完整的手动compaction，如何预估可以回收的空间？
A: 没有简单的方法可以准确的预测，尤其是在有压缩过滤器的情况下。
   如果数据库大小是稳定的，那么DB属性 "rocksdb.estimate-live-data-size" 是最好的预估。
 
Q: 快照、检查点、备份之前有什么区别？
A: 快照   是一个逻辑的概念。用户可以使用程序接口查询数据，但是底层压缩仍然重写现有文件。
   检查点 将使用相同的环境创建所有数据库文件的物理镜像。如果可以使用文件系统硬链接创建镜像文件，则此操作非常便宜。
   备份   可以将物理数据库文件移动到另一个环境（如HDFS）。备份引擎还支持不同备份之间的增量拷贝。

Q: 我应该使用哪种压缩类型？
A: 从所有级别的LZ4开始(如果没有可用的LZ4,就使用Snappy)，以获得良好的性能。如果您想进一步减少数据大小，请尝试在最后一层使用Zlib。
 
Q: 如果没有删除或覆盖键，是否需要压缩？
A: 即使不需要清除过滤数据，也需要压缩以确保读取性能。
 
Q: 在写入下列选项之后 option.disableWAL=true, 我用options.sync=true配置 写了另一条记录。它是否也会保留前面的写入？
A: 不是。程序崩溃后，用option.disableWAL=true将丢失，如果它们没有刷新到SST文件。
 
Q: options.target_file_size_multiplier 有用吗？
A: 这是一个很少使用的功能。例如，您可以使用它来减少SST文件的数量。
 
Q: 能否区分由RocksJava抛出的异常类型？
A: 是的。对于每一个与RocksDB相关的异常，RocksJava都会抛出RocksDBException异常。
 
Q: 我观察到突发写I/O，我怎么才能把他消去呢？
A: 尝试使用速率限制器：https://github.com/facebook/rocksdb/blob/v4.9/include/rocksdb/options.h#L875-L879
 
Q: 我可以在不重新打开数据库的情况下更改压缩过滤器吗？
A: 不支持。但是，您可以通过实现返回不同压缩过滤器的CompactionFilterFactory来实现它。
 
Q: iterator迭代器 Next()的性能是否与Prev()相同?
A: 反向迭代器性能通常比正向迭代器性能差很多。原因有很多：
   (1)数据块中的增量编码对Next()更友好;
   (2)memtable中使用的跳转列表是单向的，因此Prev()是另一个二分查找;
   (3)为Next()内部键顺序优化。
 
Q: 一个数据库可以支持多少个列族？
A: 用户应该能够运行至少数干个列族而不会看到任何错误。然而，大多的列族通常不能很好地执行。
   我们不建议用户使用超过几百个列族。
 
Q: RocksDB能告诉我们数据库中键的总数吗？或者一个范围内键的总数？
A: RocksDB 可以通过 DB属性 "rocksdb.estimate-num-keys" 预估键的数量。
   注意，当存在合并操作符、现有键被覆盖或删除不存在的键时，这种预估可能很不准确。
   
   预估范围内键总数的最佳方法是，首先通过盗用DB::GetApproximateSizes()预估范围的大小，然后根据该方法预估键的数量。
 
Q: RocksDB 修复: 我什么时候可以使用它？最佳实践？
A: 检查 https://github.com/facebook/rocksdb/wiki/RocksDB-Repairer
 
Q: 如果我想从rocksdb检索10个键，那么批处理它们并使用 MultiGet()与发出10个Get()调用相比，哪个更好？
A: 性能相似。MultiGet()从相同的一致视图读取数据，但速度并不快。
 
Q: 我应该如何实现多个数据库 分片/分区。
A: 可以从每个 分片/分区 使用一个RocksDB数据库开始。
 
Q: 块缓存的默认值是多少？
A: 8MB
 
Q: 如果我有多个列族，并且调用没有列族句柄的DB函数，结果会是什么？
A: 它将只操作默认的列族。
 
Q: 数据库操作由于空间不足而失效。我怎样才能让自己放松?
A: 首先清理一些空闲的空间。然后需要重新启动数据库使其恢复正常。
   现在没有一种方法可以在不重新启动数据库的情况下解决。
 
Q: 我可以跨多个线程重用ReadOptions、WriteOptions等吗？
A: 只要它们是常量，您就可以自由地重用它们。
 
Q: 我可以重用DBOptions或ColumnFamilyOptions来打开多个DB或列族吗?
A: 可以。在内部，RocksDB总是复制这些选项。因此您可以自由地更改它们并重用这些对象。
 
Q: RocksDB支持组提交吗？
A: 支持。多个线程发出的多个写请求可以组合在一起。其中一个线程在一个写请求中为这些写请求写WAL日志。如果配置好了，则一次fsync。
 
Q: 是否可以只扫描/迭代键？如果是，这比加载键和值更有效吗？
A: 不。通常效率不高。RocksDB的值通常与键一起内联存储。
   当用户遍历键时，这些值已经加载到内存中，因此跳过这些值不会节省多少。
   在BlobDB中，键和大值分别存储，因此只迭代键可能有好处，但是目前还不支持。我们可能会在未来增加支持。
 
Q: 事务对象是线程安全的吗？
A: 不，不是。您不能同时向同一事务发出多个操作。（当然，您可以并行执行多个事务，这正是该特定的重点）
   
   
Q: 迭代器移动键/值后，那些键/值所指向的内存是否仍然保留？
A: 不，它们可以被释放。除非您设置了ReadOptions.pin_data = true ，您的设置支持该特性。
 
Q: 如何预估数据库中索引和过滤块的总大小？
A: 对于离线数据库，"sst_dump --show_properties --command=none" 将显示特定sst文件的索引和过滤器大小。可以对所有DB求和。
   对于正在运行的DB，可以从DB属性 "kAggregatedTableProperties"获取。或者调用DB::GetPropertiesOfAllTables() 并总结单个文件的索引和过滤块大小。

Q: 我可以通过编程从一个SST文件读取数据吗？
A: 我们现在不支持。但是可以使用sst_dump转储数据。

# 1.介绍

    RocksDB从facebook开始，作为各种存储媒体上服务器工作负载的存储引擎，最初的重点是快速存储(尤其是闪存)。
    它是一个存储键和值(K-V)的C++库。它是任意大小的字节流。它支持点查找和范围扫描，并提供不同类型的ACID担保。
    
    在可定制性和自适应性之间取得平衡。RocksDB具有高度灵活的配置设置。可以在多种环境中运行。包括纯内存、闪存、硬盘或远程存储。
    它支持各种压缩算法和良好的生产支持和调试工具。另一方面，我们还努力限制旋钮的数量，提供足够好的开箱即用性能，并尽可能使用一些自适应算法。
    
    RocksDB 从开源的LevelDB项目及Apache HBase中借鉴了重要的代码。最初的代码是从开源 leveldb 1.5版本派生出来的。
    它还基于RocksDB之前Facebook开发的代码和想法。
    
# 2.假定和目标

  性能: RocksDB主要设计点是，对于快速存储和服务器的工作负载，它应该是高性能的。
  它应该支持有效的点查找和范围扫描。它应该是可配置的，以支持高随机读取工作负载、高更新工作负载或两者的组合。
  它的架构应该支持为不同的工作负载和硬件轻松的进行权衡。
  
  生产支持: RocksDB的设计方式应该是内置对工具和实用程序的支持，这些工具和实用程序有助于在生产环境中部署和调试。
  如果存储引擎还不能自动调整应用程序和硬件。我们将提供一些参数来允许用户性能调优。
  
  兼容性：此版本的较新版本应该向后兼容，以便在升级较新版的RocksDB时不需要更改现有的应用程序。
  除非使用新提供的特性，否则现有应用程序也应该能够恢复到最近的旧版本。
  参见不同版本之间的RocksDB兼容性 (RocksDB Compatibility Between Different Releases)
  
# 3.高级体系结构

    RocksDB是一个键值存储接口的存储引擎库，其中键和值是任意的字节流。
    RocksDB按顺序组织所有数据，常见的操作有 Get(key), NewIterator(), Put(key, val), Delete(key), and SingleDelete(key)。
    
    RocksDB的三个基本结构是memtable、sstfile和logfile。
    memtable是内存中的数据结构——新的写操作被插入到memtable中，并且可以选择写入日志文件。日志文件是存储上按顺序编写的文件。
    当memtable被填满时，它被刷新到存储的sstfile中，并且可以安全地删除相应的日志文件。对sstfile中的数据进行排序是为了方便键的查找。
    
    这里更详细地描述了默认sstfile的格式。(https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)

# 4.特性

### Column Families 列族

    RocksDB 支持将数据库实例划分到多个列族。所有数据库都使用为 "default" 的列族创建，该列族用于未指定列族的操作。
    RocksDB 保证用户在列族系列中具有一致的视图，包括启用WAL或启用原子刷新时崩溃恢复后的视图。
    它还通过 WriteBatch API 支持原子跨列系列操作。
    
### Updates 更新

    Put API 向数据库插入一个键值（K-V），如果该键已经存在于数据库中，则将覆盖以前的值。 
    Write API 允许在数据库中原子插入、更新 或 删除多个键值（K-V）。
    数据库保证一次写入调用中的所有键值都将插入数据库，或者都不会插入数据库。如果数据库中已经存在这些键中的任何一个，则将覆盖以前的值。
    特殊范围删除可以删除范围中的所有键。
    
### Gets, Iterators and Snapshots 获取、迭代器和快照

    键和值被视为纯字节流，键或值的大小没有限制。
    
    GET API 允许应用程序从数据库中获取单个键值。
    MultiGet API 允许应用程序从数据库中检索一组键。通过multiget调用返回的所有keys值都是一致的。
    
    数据库中的所有数据都按逻辑顺序排列。应用程序可以指定键比较方法，该方法指定键的总顺序。迭代器API允许应用程序对数据库进行范围扫描。
    迭代器可以查找指定的键，然后应用程序可以从该点开始一次扫描一个键。迭代器API还可以用于对数据库中键进行反向迭代。
    创建迭代器时，将创建数据库的一致时间点视图。因此，通过迭代器返回的所有键都来自数据库的一致视图。
   
    Snapshots 快照API允许应用程序创建数据库的时间点视图。Get 和 Iterator 迭代器API可以用于指定的快照读取数据。
    在某种意义上，快照和迭代器都提供了数据库的时间点视图，但它们的实现是不同的，
    Short-lived/foreground 扫描最好通过迭代器完成，而 long-running/background 扫描最好通过快照完成。
    迭代器对与数据库的时间点视图相对的所有底层文件保持引用计数----在迭代器释放之前不会删除这些文件。另一方面，快照不能防止文件删除；
    相反，压缩过程理解快照的存在，并承诺永远不会删除任何现有快照中的可见的键。
    
    Snapshots 快照不会数据库重新启动进行持久化；重新加载RocksDB库(通过服务器重新启动)将释放所有预先存在的快照。
    
### Transactions 事务

    RocksDB支持多操作事务， 它支持乐观锁和悲观锁。请查阅 Transactions（https://github.com/facebook/rocksdb/wiki/Transactions）
    
### Prefix Iterators 前缀迭代器

    大多数 LSM-Tree 引擎不能支持有效的RangeScan API ，因为它需要查看多个数据文件。但是，大多数应用程序不会对数据库中的键范围进行纯随机扫描；
    相反，应用程序通常在键前缀内进行扫描。RocksDB 充分利用了这一点。
    应用程序可以配置prefix_extractor 来指定键前缀。RocksDB使用它存储每个键前缀的bloom.
    指定前缀(通过ReadOptions) 的迭代器将使用这些bloom位，以避免查看不包含具有指定键前缀的键的数据文件。

### Persistence 持久性

    RocksDB 有一个Write Ahead Log WAL预写日志。所有put都存储在称为memtable的内存缓冲区中。
    也可以选择插入WAL。在重新启动时，它将重新处理记录在日志中的所有事务。
    
    可以将WAL配置为存储在与存储SST文件的目录不同的目录中。对于您可能希望将所有数据文件存储在非持久性快速存储器中的情况，这是必要的。
    同时，通过将所有事务日志放在较慢但持久的存储中，可以确保没有数据丢失。
    
    每个Put都有一个通过 WriteOptions 设置的标志，该标志指定是否应该将Put插入事务日志。
    WriteOptions 还可以指定是否在声明提供put之前向事务日志发出同步调用。
    
    在内部，RocksDB使用批处理提交机制将事务批处理到日志中，这样它就可以使用一个同步调用提交多个事务。

### Data Checksuming 数据校验

    RocksDB 使用校验和检测存储中的损坏。这些校验和用于每个SST文件块(通常大小在4K到128K之间)。一个块，一旦写入存储器，就永远不会被修改。
    RocksDB 动态检测对校验和计算的硬件支持，并在可用时利用这种支持。

### Multi-Threaded Compactions 多线程压缩合并

    如果应用程序覆盖已存在的key，则需要使用Compactions压缩合并来删除同一key的多个副本。合并还处理key键的删除。
    如果配置得当，可以在多个线程中进行压缩合并。
    
    整个数据库存储在一组sstfiles文件中。当memtable被填满时，它的内容被写到0级(L0)的文件中。
    当将memtable中的重复键和覆盖键刷新到L0中的文件时，RocksDB删除了它们。有些文件会定期读入并合并成更大的文件---这称为compaction压缩合并

    LSM数据库的总体写吞吐量直接取决于compaction压缩合并发生的速度，特别是当数据库存储在诸如SSD或RAM之类的快速存储中时。可以将RocksDB配置为从多个线程发出并发压缩请求。
    可以观察到，与单线程压缩合并相比，当数据库位于SSD上时，使用多线程压缩合并，持续写速率可能会增加多大10倍。

### Compaction Styles 压缩合并类型

    通用式压缩合并对全排序的运行进行操作，这些运行要么是L0文件，要么是L1+以上的整个级别。
    压缩合并选择一些按时间顺序彼此相邻的已排序的运行，并将他们压缩合并回一个新的已排序的运行。
    
    Level式合并在数据库的多个Level中存储数据。最近的数据存储在L0中，最老的数据存储在Lmax中。
    L0中的文件可能有重叠的键，但是其他层中的文件没有。压缩合并过程在Ln中选择一个文件，并在Ln+1中选择所有重叠的文件，然后用Ln+1中的新文件替换它们。
    与Level式合并相比，通用式压缩合并通常导致较低的写放大，但空间和读取放大。
 
    FIFO式压缩在过时时删除最旧的文件，并可用与类似缓存的数据。
    
    我们还允许开发人员使用定制的合并策略进行开发和试验。因此，RocksDB有适当的钩子来关闭内置的压缩合并算法。并有其他API来允许应用程序操作自己的压缩合并算法。
    disable_auto_compression 选项 (如果设置)禁用本机压缩合并算法。
    
    GetLiveFilesMetaData API允许外部组件查看数据库中的每个数据文件，并决定合并和压缩哪些数据文件。
    调用 CompactFiles 来压缩合并您想要的文件。
    DeleteFile API允许应用程序删除被认为已过时的数据文件。

### Metadata storage meta元数据存储

    数据库中的 MANIFEST 文件记录数据库的状态。
    Compaction 合并过程将添加新文件并从数据库中删除现有文件，并通过将这些操作记录在 MANIFEST 文件中使其持久化。

### Avoiding Stalls  停顿(写入优化)

    后台compaction线程还用于将memtable内容刷新到存储中的文件。如果所有后台compaction线程都忙于执行长时间运行的compaction，那么突然爆发的写操作会很快填满memtable，从而停止新的写操作。
    这种情况可以通过配置rocksdb来避免，以保留一组专门为将memtable刷新到存储而显式保留的compaction。

### Compaction Filter 压缩合并过滤器

    有些应用程序可能希望在Compaction处理key。例如，具有生存时间(TTL)固有支持的数据库可能会删除过期的key。这可以通过应用程序定义的压缩合并过滤器来完成。
    如果应用程序希望连续删除早于特定时间的数据，则可以使用压缩合并筛选器删除已过期的记录。
    RocksDB压缩合并过滤器控制应用程序修改键的值，或作为压缩合并过程的一部分完全删除键。
    例如，作为压缩合并的一部分，应用程序可以连续运行数据清理程序。

### ReadOnly Mode 只读模式

    数据库可以以只读模式打开，在这种模式下，数据库保证应用程序不能修改数据库中的任何内容。这将导致更高的读取性能，由于经常遍历的代码路径，因此完全避免了锁。

### Database Debug Logs 数据库调试日志

    默认情况下，RocksDB将详细的日志写入名为LOG*的文件，它们主要用于调试和分析运行中的系统。
    用户可以选择不同的日志级别。可以将此日志配置为以指定的周期滚动。
    日志记录接口是可插入的，用户可以插入不同的日志记录器。

### Data Compression 数据压缩

    RocksDB 支持lz4、ztsd、snappy、zlib和lz4_hc压缩，以及Windows下xpress。
    RocksDB 可以配置为支持底层数据的不同压缩算法，底层数据占数据的90%。典型的安装可能会为最底层配置ZSTD(如果不可用，则配置zlib)，而为其他级别配置LZ4(如果不可用，则配置Snappy)。
    查阅：压缩(https://github.com/facebook/rocksdb/wiki/Compression)

### Full Backups and Replication 全备份 和 复制

    RocksDB 提供了一个备份引擎 BackupableDB。你可以在这里阅读更多：(https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F)
    
    RocksDB 本身不是复制的，但它提供了一些帮助函数，使用户能够在上面 或 RocksDB上实现他们的复制系统。请参阅 (https://github.com/facebook/rocksdb/wiki/Replication-Helpers)

### Support for Multiple Embedded Databases in the same process 在同一进程中支持多个嵌入式数据库

    RocksDB 的一个常见用例是 应用程序将其数据集天生地划分为逻辑分区或碎片。该技术有利于应用程序负载平衡和故障的快速恢复。
    这意味着一个服务器进程应该能够操作多个RocksDB数据库。这是通过一个名为Env的环境对象完成的。其中，线程池与Env关联。如果应用程序希望在多个数据库实例之间共享一个公共线程池(用于后台压缩)，那么它应该使用相同的Env对象来打开这些数据库。
    类似地，多个数据库实例可能共享一个块缓存。

### Block Cache -- Compressed and Uncompressed Data 块缓存---压缩和未压缩的数据

    RocksDB 使用LRU缓存来为块提供读取服务。块存储被划分为两个单独的缓存:第一个缓存未压缩快，第二个缓存RAM中的压缩块。
    如果配置了压缩块缓存，用户可能希望启动direct I/O（直接IO），以防止OS页面缓存对相同的压缩数据进行双缓存。

### Table Cache 表缓存

    表缓存是缓存打开的文件描述符的构造。这些文件描述符用户sstfiles。应用程序可以指定表缓存的最大大小，或者将RocksDB配置为始终保持所有文件打开，
    以获得更好的性能。

### I/O Control I/O控制

    RocksDB允许用户以不同的方式从和到SST文件配置IO。用户可以启用direct IO(直接IO), 以便RocksDB就可以完全控制I/O和缓存。
    另一种方法是利用一些选项，允许用户提示应该如何执行I/O。他们可以建议rocksdb在要读取的文件中调用fadvise，在要追加的文件中调用定期范围同步，或者启用直接I/O。查阅:IO(https://github.com/facebook/rocksdb/wiki/IO)

### Stackable DB 

    RocksDB有一个内置的包装器机制，可以作为代码数据库内核智商的一层添加功能。
    该功能由StackableDB API封装。例如，实时功能是由StackableDB实现的，并不是核心RocksDB API的一部分。这种方法保持代码模块化和干净。
    
### Memtables   
  
#### Pluggable Memtables 插件式Memtables

     RocksDB 的memtable的默认实现是一个skiplist, skiplist是一个排序的集合，当工作负载将写与范围扫描交叉时，这是一个必要的构造。
     然而，有些应用程序不交错写和扫描，有些应用程序根本不执行范围扫描。对于这些应用程序，排序集可能无法提供最佳性能。
     因此，RocksDB的memtable是可插入的。提供了一些替代实现。三个memtable是库的一部分；一个skiplist memtable、一个vector memtable和一个前缀-hash memtable。
     向量memtable适用于将数据批量加载到数据库中。每次写操作都会在向量的末尾插入一个新元素；当需要刷新memtable以存储向量中的元素时，将对向量中的元素进行排序并将其写入L0中的文件中。
     前缀hash memtable允许高效地处理get、put和scans-within-a-key-prefix。虽然没有将memtable的可插拔性作为公共API提供，
     但是应用程序可以在私有分支中提供自己的memtable实现。
     
#### Memtable Pipelining  Memtable管道

     RocksDB支持为数据库配置任意数量的memtable，当memtable被填满时，它就变成了一个不可变的memtable，后台线程开始将其内容刷新到存储器中。同时，新的写操作继续累积到新分配的memtable中。
     如果新分配的memtable被填满，那么它也会呗转换为不可变的memtable，并插入到刷新管道中。
     后台线程继续将所有管道化的不可变memtable刷新到存储器。这种管道增加了RocksDB的写吞吐量，特别是当它在慢速存储设置上运行时。

#### Garbage Collection during Memtable Flush  内存表刷新期间的垃圾回收

     当将memtable刷新到存储中时，将执行一个内联压缩过程。垃圾和垃圾填充物的去除方法是一样的。
     从输出蒸汽中删除相同键的重复更新。类似地，如果前面的put呗后面的delete隐藏，那么put就根本不会写入输出文件。
     对于某些工作负载，该特性大大减小了存储和写入数据的大小。

### Merge Operator Merge操作

     RocksDB本身支持三种类型的记录:Put记录、Delete记录 和 Merge记录。当压缩过程中遇到合并记录时，它调用应用程序制定的方法Merge操作符。合并可以将多个Put和Merge记录合并到一个记录中。
     这个强大的特性允许通常执行read-modify-writes操作的应用程序完全避免读操作。
     它允许应用程序将操作意图记录为合并记录，而RocksDB压缩过程将该意图惰性地应用于原始值。
     这个特性在Merge操作符中有详细描述。(https://github.com/facebook/rocksdb/wiki/Merge-Operator)

# 5.工具

    在生产中，有许多有趣的工具用于支持数据库。sst_dump实用程序将所有键值以及其他信息转储到一个sst文件中。
    ldb工具可以放置、获取和扫描数据库的内容。
    ldb还可以转储清单的内容，它还可以用来更改已配置的数据库级别的数量。
    有关详细信息：请参阅 管理和数据访问工具 (https://github.com/facebook/rocksdb/wiki/Administration-and-Data-Access-Tool)

# 6.测试

    有一组单元测试测试数据库的特特性。make check命令运行所有单元测试。单元测试触发了RocksDB的特定特性，并不是设计用于大规模测试数据正确性。
    db_stress测试用于按比例验证数据的正确性。

# 7.性能

    RocksDB 性能通过一个名为 db_bench的实用程序进行基准测试。db_bench是RocksDB源代码的一部分。
    这里描述了一些实用闪存的典型工作负载和性能结果。（https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks）
    您还可以在这里找到用于内存工作负载的RocksDB性能结果。（https://github.com/facebook/rocksdb/wiki/RocksDB-In-Memory-Workload-Performance-Benchmarks）

注明：Compactions的含义: 压缩合并 及 数据整理。
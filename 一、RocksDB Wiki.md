# 欢迎来到RocksDB

RocksDB 是一个带有key/value接口类存储引擎，其中键和值是任意的字节流。它是一个c++库。
它是在facebook上基于LevelDB开发的，并为LevelDB api提供向后兼容的支持。

RocksDB 支持各种存储硬件，最初的焦点是快速flash，它使用一个日志结构的数据库引擎进行存储，
完全用C++编写，并有一个名为RocksJava的Java包装器。可查阅 RocksJava(https://github.com/facebook/rocksdb/wiki/RocksJava-Basics)基础知识)

RocksDB 可以适应多种生产环境，包括纯内存、闪存、硬盘或远程存储。当RocksDB不能自动适应时，提供了高度灵活的配置设置，
允许用户对其进行调优，它支持各种压缩算法和良好的生产支持和调试工具。

### 功能

    * 专为希望在本地或远程存储系统上存储数兆字节(TB)数据的应用服务器而设计。
    * 优化存储中小尺寸的键值在快速存储--闪存设备或内存。
    * 它在多核处理器上运行良好。
    
### 不在LevelDB中的特性

    RocksDB 引入了许多新的主要特性。查看不在LevelDB中的特性列表 (https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB)

### 入门指南

    有关完整的目录，请参见右侧的侧栏。大多数读者都希望从开发人员指南的概述 和 基本操作部分开始(https://github.com/facebook/rocksdb/wiki/Basic-Operations）。根据设置选项和基本调优获得初始选项设置（https://github.com/facebook/rocksdb/wiki/Setup-Options-and-Basic-Tuning)。也检查RocksDB FAQ(https://github.com/facebook/rocksdb/wiki/RocksDB-FAQ)。
    还有一个针对高级RocksDB用户的RocksDB调优指南（https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide）

### 报告错误并寻求帮助

    如果您遇到任何问题，请使用这些指南报告错误并寻求帮助。(https://github.com/facebook/mysql-5.6/wiki/Reporting-bugs-and-asking-for-help)
 
### Blog 博客
   
    查看我们的博客 rocksdb.org/blog(http://rocksdb.org/blog)

### 项目历史

    RocksDB变更历史 (http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)
    引擎盖下: 构建和开源RocksDB (https://www.facebook.com/notes/facebook-engineering/under-the-hood-building-and-open-sourcing-rocksdb/10151822347683920)
    
### 链接

    例子 (https://github.com/facebook/rocksdb/tree/master/examples)
    官方博客 (http://rocksdb.org/blog/)
    Stack Overflow: RocksDB (https://stackoverflow.com/questions/tagged/rocksdb)
    Talks (https://github.com/facebook/rocksdb/wiki/Talks)

### 联系方式

    公共开发讨论组（https://www.facebook.com/groups/rocksdb.dev/）

### 新的RocksDB生产文档

    RocksDB 生产文档（https://shahmeeramir.com）
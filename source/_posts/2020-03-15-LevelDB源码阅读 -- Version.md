---
title: LevelDB源码阅读 -- Version  
date: 2020-03-15 23:31:30
categories:
- 源码阅读
tags: [源码阅读, leveldb]
---

![](https://z3.ax1x.com/2021/05/04/gmjpgP.jpg)

<!-- more -->

本文主要介绍LevelDB中的Version及其相关的结构和流程。

理解Version对理解LevelDB中compaction有很重要的作用，可以说不理解Version就无法理解Compaction。Version和Compaction互相联系，相互作用。Version，直观上理解就是版本，也就是说LevelDB中数据是可能是多版本共存的，所以在介绍Version具体相关的数据结构之前，我们先看一下MVCC。

## MVCC

Multi-version Cocurrent Control，多版本并发控制。MVCC 技术最初也是在数据库

系统中被提出，但这种思想并不局限于单机的分布式系统，在分布式系统中同样有效。

顾名思义，MVCC 即多个不同版本的数据实现并发控制的技术，其基本思想是为每次事务生成一个新版本的数据，在读数据时选择不同版本的数据即可以实现对事务结果的完整性读取。在使用 MVCC 时，每个事务都是基于一个已生效的基础版本进行更新，事务可以并行进行，从而可以产生一种图状结构。

![](https://z3.ax1x.com/2021/05/04/gmjpgP.jpg)

如上图所示，基础数据的版本为1，同时产生了两个事务：事务A 与事务B。这两个事务都各自对数据进行了一些本地修改（这些修改只有事务自己可见，不影响真正的数据），之后事务A 首先提交，生成数据版本2；基于数据版本2，又发起了事务C，事务C 继续提交，生成了数据版本3；最后事务B 提交，此时事务B 的结果需要与事务C 的结果合并，如果数据没有冲突，即事务B 没有修改事务A 与事务C 修改过的变量，那么事务B 可以提交，否则事务 B 提交失败。

MVCC 的流程过程非常类似于SVN 等版本控制系统的流程，或者说SVN 等版本控制系统就是使用的 MVCC 思想。

事务在基于基础版本做修改时，为了不影响真正的数据。通常有两种做法：

1. 将基础版本的数据全部拷贝出来之后再修改，SVN就使用了这个办法，SVN check out就是拷贝的过程
2. 每个事务中只记录更新操作，不记录完整的数据，读取数据的再将更新操作应用于基础版本的数据从而得到更新后的数据

LevelDB中的sstable的MVCC采用的上述的第2种方法，下文介绍LevelDB中Version相关内容。

## Version、VersionEdit、VersionSet

### 关系介绍

* Version字面意思就是代表某一版本
* VersionEdit就是指版本变化（一个版本到另一个版本之间的更新操作）
* VersionSet看字面意思就是版本的集合，确实也是系统中Version的集合

上文提到了LevelDB也有SSTable的MVCC，其具体实现就是通过Version、VersionEdit以及VersionSet实现的。Version代表一个基础版本，VersionEdit就是记录更新操作当中的各种变更，VersionSet则是管理了当前系统中所有的版本。也就是说有如下公式：

**Version(N) + VersionEdit = Version(N+1)**

在LevelDB的实现中，为方便将VersionEdit变更应用于Version之上，实现了VersionSet::Builder以及相关的方法。不过有个问题，如果有多次变更操作产生了多次VersionEdit，然后将这多个VersionEdit累积应用与基础Version版本，将会得到多个Version，如果只需要最终的一个Version或者某几个Version，那这个过程岂不是产生了很多中间版本。为了达到不产生这些中间版本的目的，VersionSet Builder中提供了一个Apply方法，可以把多次变更累积到一个VersionEdit中，然后将该VersionEdit应用于(VersionSet Builder提供的方法为SaveTo)基础版本Version就可以直接得到最终版本。



### 数据结构

本文只介绍下关键的数据结构成员及关键方法，并不会列出所有的数据成员，详细数据结构请参考源码。

* Version的主要数据结构

```

VersionSet* vset_;  // 当前版本属于的VersionSet

Version* next_;     // 在VersionSet双向链表中的下一个版本

Version* prev_;     // 在VersionSet双向链表中的前一个版本

int refs_;          // 当前版本存有的引用计数

// 每一层所拥有的文件列表

std::vector<FileMetaData*> files_[config::kNumLevels];

// 基于统计访问次数选定的将要进行下一次compaction的文件*

FileMetaData file_to_compact_;

int file_to_compact_level_;

// Level that should be compacted next and its compaction score.

// Score < 1 means compaction is not strictly needed.  These fields

// are initialized by Finalize().

double compaction_score_;

int compaction_level_;

```

从Version的主要数据结构可以看到，当前版本中包含属于当前版本的所有的文件，以及哪个文件将要用于compaction，还记录了compaction_score_以及compaction_level_（将要用于压缩的那一层，其compaction_score是最大的，注释中也提到了，score < 1的情况下，compaction是不严格要求的, 也就是说这种情况可以不进行compaction）。此外，Version的数据结构中还记录上一个版本prev_以及下一个版本next_。

* VersionEdit主要数据结构

```

std::vector<std::pair<int, InternalKey>> compact_pointers_;

DeletedFileSet deleted_files_;

std::vector<std::pair<int, FileMetaData>> new_files_;

```

上面三个是VersionEdit中最重要的成员，每一个compact_pointers_表示某一层的要开始进行compact的InternalKey，比如在实际要进行compaction时，通过该变量可以知道在当前层（pair当中的第一个变量）从哪一个Key开始进行compaction，没有这个信息的话，就不知道从哪个key开始进行compaction，总不能每次都从第一个key开始吧。

deleted_files_则表示要删除哪些文件。

new_files_则表示新增哪些文件。

* VersionSet主要数据结构

```

Versiondummy_versions_; // Head of circular doubly-linked list of versions.

Version*current_;      // == dummy_versions_.prev_

// Per-level key at which the next compaction at that level should start.

// Either an empty string, or a valid InternalKey.

std::stringcompact_pointer_[config::kNumLevels];

```

dummy_versions_是双向循环链表的头，也就是说VersionSet是一个双向循环连接，所有存在的Version都在这个双向循环链表上。

dummy_versions_.prev_指向当前版本current_。

compact_pointer_也是表明每一层进行下一次compaction时应该开始的Key，这个要么是个空字符串，要么是一个有效的InternalKey。

通过上面VersionEdit的主要数据结构可以看出，在LevelDB的MVCC是SSTable的MVCC，VersionEdit主要是记录了SSTable文件的变化（在compaction时生成新的SSTable，完成compaction之后删除旧的SSTable）。所以严格的来说，这里的MVCC并不是读取某一个数据多版本，当然由于compaction前同一个key在不同文件中确实有存在多个版本的问题，比如刚开始设置key1的值为1，然后一直有写请求，后面又将key1的值设为了2。那么这系统里面是存在两个版本的key1的value值的，我们读的时候，是会优先读到2的这个值的，也可以通过设置来读取到1，不过在进行compaction的时候会进行合并，合并之后就不会读到1了。

### 主要方法及流程

### 更多问题

1.既然存在多版本，那什么时候Version会发生变化呢？

    答：当SSTable文件发生变化时则会产生新的Version

2.SSTable产生变化的触发条件是什么？

 答：（1）将Immutable转为SSTable时（minor compaction）

     （2）当后台开始进行Compact时 （major compaction）

3.Version产生的流程

 答：

* Memtable转为新的SSTable或者进行Compact之后SSTable产生变化时调用**VersionSet::LogAndApply()**产生新的Version。
* leveldb上电还原Version过程，调用接口**VersionSet::Recover()**。
* leveldb异常损坏，修复levelDB过程，调用接口RepairDB()产生新的Version。


4.Manifest丢失或者损坏，LevelDB会丢数据吗？

  答：不会

5.Manifest丢失或者损坏，如何恢复LevelDB数据？

 答：使用python-leveldb，通过如下手段可以修复Leveldb

```plain
import leveldb
ret ＝ leveldb.RepairDB('/data/mon.iecvq/store.db')
```


6.为什么MANIFEST损坏或者丢失之后，依然可以恢复出来？LevelDB如何做到的？

答：对于LevelDB而言，修复过程如下：

* 首先处理log，这些还未来得及写入的记录，写入新的.sst文件
* 扫描所有的sst文件，生成元数据信息：包括number filesize， 最小key，最大key
* 根据这些元数据信息，将生成新的MANIFEST文件。

第三步如何生成新的MANIFEST？ 因为SSTable文件是分level的，但是很不幸，我们无法从名字上判断出来文件属于哪个level。第三步处理的原则是，既然我分不出来，我就认为所有的sstale文件都属于level 0，因为level 0是允许重叠的，因此并没有违反基本的准则。

当修复之后，第一次Open LevelDB的时候，很明显level 0 的文件可能远远超过4个文件，因此会Compaction。 又因为所有的文件都在Level 0 这次Compaction无疑是非常沉重的。它会扫描所有的文件，归并排序，产生出level 1文件，进而产生出其他level的文件。

从上面的处理流程看，如果只有MANIFEST文件丢失，其他文件没有损坏，LevelDB是不会丢失数据的，原因是，LevelDB既然已经无法将所有的数据分到不同的Level，但是数据毕竟没有丢，根据文件的number，完全可以判断出文件的新旧，从而确定不同sstable文件中的重复数据，which是最新的。经过一次比较耗时的归并排序，就可以生成最新的LevelDB。

上述的方法，从功能的角度看，是正确的，但是效率上不敢恭维。Riak曾经测试过78000个sstable 文件，490G的数据，大家都位于Level 0，归并排序需要花费6 weeks，6周啊，这个耗时让人发疯的。

Riak 1.3 版本做了优化，改变了目录结构，对于google 最初版本的LevelDB，所有的文件都在一个目录下，但是Riak 1.3版本引入了子目录， 将不同level的sst 文件放入不同的子目录：

```plain
sst_0
sst_1
...
sst_6
```
有了这个，重新生成MANIFEST自然就很简单了，同样的78000 sstable文件，Repair过程耗时是分钟级别的。

7.compaction触发的时机？

答：主要有如下三个触发的时机

* **容量触发Compaction：**每个Version在其生成的时候会初始化两个值compaction_level_、compaction_score_，这两个值记录了当前Version最需要进行Compaction的Level，以及其需要进行Compaction的紧迫程度，score大于1被认为是需要马上执行的。我们知道每次文件信息的改变都会生成新的Version，所以每个Version对应的这两个值初始化后不会再改变。level0层compaction_score_与文件数相关，其他level的则与当前层的文件总大小相关。这种区分的必要性也是显而易见的：每次Get操作都需要从level0层的每个文件中尝试查找，因此控制level0的文件数是很有必要的。同时Version中会记录每层上次Compaction结束后的最大Key值compact_pointer_，下一次触发自动Compaction会从这个Key开始。容量触发的优先级高于下面将要提到的Seek触发。
* **Seek触发Compaction：**Version中会记录file_to_compact_和file_to_compact_level_，这两个值会在Get操作每次尝试从文件中查找时更新。LevelDB认为每次查找同样会消耗IO，这个消耗在达到一定数量可以抵消一次Compaction操作消耗的IO，所以对Seek较多的文件应该主动触发一次Compaction。但在引入布隆过滤器后，这种查找消耗的IO就会变得微不足道了，因此由Seek触发的Compaction其实也就变得没有必要了。
* **手动Compaction：**LevelDB提供了外部接口CompactRange，用户可以指定触发某个Key Range的Compaction，LevelDB默认手动Compaction的优先级高于两种自动触发。


### 参考资料

《分布式系统原理介绍》

[leveldb之MANIFEST](https://bean-li.github.io/leveldb-manifest/)

[图解leveldb version相关结构及流程](https://blog.csdn.net/H514434485/article/details/108210958)

[庖丁解LevelDB之版本控制](https://catkang.github.io/2017/02/03/leveldb-version.html)


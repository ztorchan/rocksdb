# 为RocksDB添加冷热区分
本项目的目标是为RocksDB添加热点数据识别的功能，将热数据和数据在内存的MemTable和存储中的SSTable均分离存储，以提高数据读取的速度。具体理论思路参考论文，本文主要对其在RocksDB上实现的工程思路和细节进行探讨和记录。
<!-- 已经是第5天了，早该开始记录了，这几天源码和思路看了忘，忘了看，文件又多的要死，不停重复导致效率极低 -->

## 源码要点
- **`Put`操作的主要调用流程：**
  1. 应用程序调用`DBImpl::Put()`，提供参数`WriteOption`、`ColumnFamily`、`Key`和`Value`。若不指定列族，则会使用默认列族`DefaultColumnFamily()`。随后`DBImpl::Put()`会调用其基类的方法``DB::Put()`。
  2. `DB::Put()`会构造一个`WriteBatch`对象，通过`WriteBatch::Put()`对请求进行封装。更进一步地说，`WriteBatch::Put()`会先调用用`WriteBatchINternal::GetColumnFamilyIdAndTimestampSize()`获取时间戳和列族ID，**再调用`WriteBatchInternal::Put()`将请求信息记录在`WriteBatch`里**。（对于所有`XXInternal`类可以视为`XX`类的工具类，一般仅提供static方法）。
  3. **（重点）**`WriteBatchInternal::Put()`会对请求进行编码，将其放入`WriteBatch::rep_`里，具体编码形式如下：
    ``` C++
    /* rep_ */
    WriteBatch::rep_ :=
      sequence: fixed64     // rep_长度
      count: fixed32        // record个数
      data: record[count]   // record信息
    
    /* record格式：record类型 + 相应信息*/
    record := 
      kTypeValue varstring varstring
      kTypeDeletion varstring
      kTypeSingleDeletion varstring
      kTypeRangeDeletion varstring varstring
      kTypeMerge varstring varstring
      kTypeColumnFamilyValue varint32 varstring varstring
      kTypeColumnFamilyDeletion varint32 varstring
      kTypeColumnFamilySingleDeletion varint32 varstring
      kTypeColumnFamilyRangeDeletion varint32 varstring varstring
      kTypeColumnFamilyMerge varint32 varstring varstring
      kTypeBeginPrepareXID
      kTypeEndPrepareXID varstring
      kTypeCommitXID varstring
      kTypeCommitXIDAndTimestamp varstring varstring
      kTypeRollbackXID varstring
      kTypeBeginPersistedPrepareXID
      kTypeBeginUnprepareXID
      kTypeWideColumnEntity varstring varstring
      kTypeColumnFamilyWideColumnEntity varint32 varstring varstring
      kTypeNoop
    
    /* 变长string格式 */
    varstring :=
      len: varint32
      data: uint8[len]
    ```
  4. 完成封装后回到`DB::Put()`，调用`DBImpl::Write()`，进而再调用`DBImpl::WriteImpl()`开始执行写入流程。
  5. `DBImpl::WriteImpl()`大致可以概括成三个分支：(1)只写入到WAL，会跳转`WriteImplWALOnly()`；(2)Pipeline写入，会跳转`PipelinedWriteImpl()`；(3)非Pipeline写入，继续执行。关注非Pipeline写入，又分为两种情况：是否能够对Memtables多线程并发写入（通过option配置）。仅关注单线程顺序写入，方法会针对传入的`WriteBatch`构造一个`Writer`并通过`WriteThread::JoinBatchGroup()`将其添加进入`DBImpl`所持有的`WriteTread`对象中。
  6. `DBImpl::WriteImpl()`内，真正执行写入是在`WriteBatchInternal::InsertInto()`处。跳转进入`WriteBatchInternal::InsertInto()`内，构造`MemTableInserter`对象作为handler。对其进行一系列配置和验证后，调用`w(Writer)->batch(WriteBatch)->Iterate()`，传入inserter。
  7. `Iterate()`内有一个循环，对封装在`WriteBatch`对象内的操作记录逐个执行。先调用`ReadRecordFromWriteBatch()`对当前记录进行解码，得到操作类型`tag`、列族`column_family`和键值对`key, value`等信息。接着进入switch块，根据`tag`执行不同的流程。对于`Put`，将会执行`MemTableInserter::PutCF()`，在其内进而调用`MemTableInserter::PutCFImpl()`进行写入。
  8. `MemTableInserter::PutCFImpl()`首先会调用`MemTableInserter::SeekToColumnFamily()`，其内，会有`bool found = cf_mems_->Seek(column_family_id);`语句，根据列族寻找对应的`ColumnFamilyData`对象并将其设置为当前所使用的列族对象。随后`MemTableInserter::PutCFImpl()`继续执行，通过`MemTable* mem = cf_mems_->GetMemTable();`得到当前需要操作的Memtable，开始执行写入操作。
  9. 遵循项目相关源码优先的原则，对`Put`的分析暂且到此结束，后续的操作我们还不用关心。**注意**：(1)以上所提到的流程是仅仅是非常简单的主要流程，之中有非常多的细节还没有探索，等有空再琢磨；(2)虽然上述只提到了`Put`操作，但是不论是哪一种操作，它们是实现流程都大同小异，以同样的路线分析即可。
- **RocksDB几个重要类的引用关系**：
  ```C++
  // in column_family.h
  //       +----------------------+    +----------------------+   +--------+
  //   +---+ ColumnFamilyHandle 1 | +--+ ColumnFamilyHandle 2 |   | DBImpl |
  //   |   +----------------------+ |  +----------------------+   +----+---+
  //   | +--------------------------+                                  |
  //   | |                               +-----------------------------+
  //   | |                               |
  //   | | +-----------------------------v-------------------------------+
  //   | | |                                                             |
  //   | | |                      ColumnFamilySet                        |
  //   | | |                                                             |
  //   | | +-------------+--------------------------+----------------+---+
  //   | |               |                          |                |
  //   | +-------------------------------------+    |                |
  //   |                 |                     |    |                v
  //   |   +-------------v-------------+ +-----v----v---------+
  //   |   |                           | |                    |
  //   |   |     ColumnFamilyData 1    | | ColumnFamilyData 2 |    ......
  //   |   |                           | |                    |
  //   +--->                           | |                    |
  //       |                 +---------+ |                    |
  //       |                 | MemTable| |                    |
  //       |                 |  List   | |                    |
  //       +--------+---+--+-+----+----+ +--------------------++
  //                |   |  |      |
  //                |   |  |      |
  //                |   |  |      +-----------------------+
  //                |   |  +-----------+                  |
  //                v   +--------+     |                  |
  //       +--------+--------+   |     |                  |
  //       |                 |   |     |       +----------v----------+
  // +---> |SuperVersion 1.a +----------------->                     |
  //       |                 +------+  |       | MemTableListVersion |
  //       +---+-------------+   |  |  |       |                     |
  //           |                 |  |  |       +----+------------+---+
  //           |      current    |  |  |            |            |
  //           |   +-------------+  |  |mem         |            |
  //           |   |                |  |            |            |
  //         +-v---v-------+    +---v--v---+  +-----v----+  +----v-----+
  //         |             |    |          |  |          |  |          |
  //         | Version 1.a |    | memtable |  | memtable |  | memtable |
  //         |             |    |   1.a    |  |   1.b    |  |   1.c    |
  //         +-------------+    |          |  |          |  |          |
  //                            +----------+  +----------+  +----------+
  //
  ```
- **关于Flush**
  - 在`MemTable`每次被操作后（例如`Add`、`Update`等），都会调用`UpdateFlushState`来判断MemTable是否需要被Flush，更新状态`FLUSH_NOT_REQUESTED -> FLUSH_REQUESTED`。
  - 在上层操作中（例如`PutCFImpl`等），最后都会调用`CheckMemtableFull()`将那些被设置为`FLUSH_REQUESTED`的MemTable设置为`FLUSH_SCHEDULED`，并将其列族放入调度队列中。
  - MemTable切换主要过程由`SwitchMemtable()`完成，在`DbImpl::HandleWriteBufferFull()`, `DBImpl::SwitchWAL()`, `DBImpl::FlushMemTable()`, `DBImpl::ScheduleFlushes()`四个函数中有调用。
  - 符合被Flush的Immu MemTable会通过`DBImpl::SchedulePendingFlush()`放入`DBImpl::flush_queue_`。随后`MaybeScheduleFlushOrCompaction()`会调用后台刷新线程`BGWorkFlush`，最终通过`BackgroundFlush()`函数完成Flush工作。
  - 关于Flush的过程参考[Memtable flush分析](https://zhuanlan.zhihu.com/p/414145200)
## 思路
- **关于写入**
  - 在`options`中添加一个`hot_table_thred`选项，用于配置热数据记录表阈值，为0时关闭热数据表功能。默认为0.
  - 为`DBImpl`添加热数据记录表。
  - 为`ColumnFamilyData`增加`hot_mem_`，表示热数据MemTable。
  - 为`MemTableInserter`添加热数据记录表指针，用于从`DBImpl`中获得热数据记录表，即可在写入时判断key是否为热数据。
  - 为`WriteBatchInternal::InsertInto`添加一个热记录表指针参数，用于构建`MemTableInserter`。
  - 为`GetMemTable()`添加一个bool变量参数，由此来判断是否对于当前key，应当选择`mem_`或`hot_mem_`。
- **关于Flush**
  - 针对不同的Flush时机，调整两个Memtable的切换需求：
    - `DBImpl::SwitchWAL()`需要将两个Memtable都切换。
    - ......
- **关于Version和SuperVersion**
  - ...
## TODO
- 修改`PutCFImpl`及其内部相关的函数
- 对`ColumnFamilyData`内部的所有函数看一遍，重点关注与MemTable和LogNumber有关的函数。
- 开始对查询部分的改动。
- 开始对Flush相关的剩下三个函数进行修改。
- 开始查看Compaction相关的源码。
## Change Log
- **2022.10.15**：开始记录CHANGE_LOG.md。尝试为编解码和`Put`相关的函数添加了bool变量`is_hot`。但也从而引出了一系列问题。
- **2022.10.16**：在`DBImpl`层判断key是否为热数据不是一个好方法，会造成大量的接口改动。不再将其编码进入`WriteBatch`，而是传递`DBImpl`的热数据表指针，由`MemTableInserter`做热数据判断。
- **2022.10.17**；梳理了一下关于SwitchMemTable和Flush的流程。开始对写入过程的改动，明天应该能完成。
- **2022.10.18**：`ColumnFamilyData`里的`MemTableList`保持一个就好了，去掉`hot_imm_`，由`MemTable`的`is_hot_`来区分是否为热数据。开始修改Flush相关的函数，添加了`SwitchHotTable`函数用于切换热MemTable。尝试完成了`DBImpl::SwitchWAL()`的修改。
## 存在问题
- RocksDB的版本控制机制`Version`和`SuperVersion`等暂且还没完全弄清楚，可能也会影响到项目改动。
- Compaction还没开始看。
- ......
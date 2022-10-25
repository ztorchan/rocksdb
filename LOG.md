# 为RocksDB添加冷热区分
> 先前的思路是，在内存设置MemTable和HotMemTable分别存放冷热数据，随后分别Flush进入不同的SST。但是这样是有问题的：
> （1）可能会出现冷热MemTable有相同的key，而且冷MemTable里的数据更加新的情况。这样按照HotMemTable -> MemTable的读取策略就会有问题，会错误地读出旧数据。其实可以两个MemTable都读出数据并比较LSN选择较新的那个，但这样冷热MemTable的区分毫无意义。
> （2）由于RocksDB的ImmuMemTable是一个MemTableList，在Flush时会首先对其中所有的MemTable做一个Compaction操作，因此分冷热MemTable也不契合RocksDB的逻辑。
> 综上考虑，不再对MemTable进行冷热区分，而是在Flush时再进行分离，放入不同的SST里。

本项目的目标是为RocksDB添加热点数据识别的功能，将热数据和数据在内存的MemTable和存储中的SSTable均分离存储，以提高数据读取的速度。具体理论思路参考论文，本文主要对其在RocksDB上实现的工程思路和细节进行探讨和记录。  

## 源码要点
### 关于Flush
- 先忽略Flush被调用的地方。
- 需要关注的Flush主流程调用栈：`FlushMemTableToOutputFile()` -> `FlushJob::Run()` -> `WriteLevel0Table()` -> `BuildTable()`
- 在Flush过程中会有`FileMetaData`的参与，它在数据写入SST的过程频繁变化，并在最后会被提交记录。
- `FlushJob`的构建发生于`FlushMemTableToOutputFile()`，重点关注`flush_job.PickMemTable()`、`flush_job.Run()`和`flush_job.GetCommittedFlushJobsInfo()`。
- `WriteLevel0Table()`会构建`MergingIterator`并传递给`BuildTable()`构建`CompactionIterator`。
- 注意`CompactionIterator`取出的key已经进行过编码处理，不可以直接用来判断冷热，应当用ikey.user_key。

## 思路
### 关于Flush
- 需要在整个流程里多添加一个`FileMetaData`，一共两个分别用于冷热SST的生成过程。
- 在`BuildTable()`中，每次迭代`CompactionIterator`取出Key就要结合HotTable判断其属于冷还是热数据，送入不同的处理分支。

## 目前问题和难点
- Flush过程中的两个FileMetaData那些信息是共享的哪些是独立的。
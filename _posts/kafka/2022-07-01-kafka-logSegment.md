---
layout: post
title: LogSegment
subtitle: kafka日志段管理
categories: Kafka
tags: [Kafka]
---

### 1. LogSegment

kafka中，broker需要对生产者的消息做持久化，将消息以文件的形式存储到服务器的磁盘上，当消费者来消费时，broker需要去磁盘读取文件，找到消息然后发送给消费者。Kafka对日志持久化做了精心的设计，通过分段的形式，将日志文件从一个大文件分割成一个个的LogSegment，日志的写入和读取都可以通过操作LogSegment来实现。



在kafka的log目录下，每个topic都有一个单独的文件夹，这些文件夹里面保存了日志文件，里面有三种类型的文件

1. .log文件，存储了真实的消息
2. .index文件，存储了消息的位移索引
3. .timeindex文件，存储了消息的时间戳索引

![image-20220728144057296](../../images/2022-07-01-kafka-logSegment/image-20220728144057296.png)

每个文件名20位长度，文件名是这个LogSegment最小的offset值，也就是baseOffset值。消费消息时，会根据消费的offset值决定去哪个LogSegment文件中检索。



磁盘的io是非常低效的，因此在消费者根据位移消费时，如果broker去顺序的遍历日志文件读取，那消费的效率将会非常低下，所以LogSegment为每个日志文件增加了一个位移索引文件，用跳表或者说稀疏索引来加快消息的检索。



用一张图来展示消息的检索过程:

<img src="../../images/2022-07-01-kafka-logSegment/image-20220705161822052.png" alt="image-20220705161822052" style="zoom:50%;" />

例如某个消费者要从offset=100来消费消息，会首先读取索引，用二分法定位到小于100的最大值也就是offset=92，而对应的position(物理地址)=4176,随后会从日志文件中读取4176长度后，继续往下读取，直到读取到offset=100。



### 2. append

在LogSegment.scala中，用append方法来将消息写到日志:

```scala
def append(largestOffset: Long,
             largestTimestamp: Long,
             shallowOffsetOfMaxTimestamp: Long,
             records: MemoryRecords): Unit = {
    if (records.sizeInBytes > 0) {
      val physicalPosition = log.sizeInBytes()
      if (physicalPosition == 0)
        rollingBasedTimestamp = Some(largestTimestamp)
      	ensureOffsetInRange(largestOffset)

      	// append the messages
      	val appendedBytes = log.append(records)

        if (largestTimestamp > maxTimestampSoFar) {
          maxTimestampSoFar = largestTimestamp
          offsetOfMaxTimestampSoFar =shallowOffsetOfMaxTimestamp
        }
        if (bytesSinceLastIndexEntry > indexIntervalBytes) {
          offsetIndex.append(largestOffset, physicalPosition)
          timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
          bytesSinceLastIndexEntry = 0
        }
        bytesSinceLastIndexEntry += records.sizeInBytes
      }
  }
```

大概总结一下这个方法:

1. 通过ensureOffsetInRange方法来判断位移索引是否合法
2. 调用log.append将消息写到页缓存

```scala
public int append(MemoryRecords records) throws IOException {
  if (records.sizeInBytes() > Integer.MAX_VALUE - size.get())
  throw new IllegalArgumentException("...");
  int written = records.writeFullyTo(channel);
  size.getAndAdd(written);
  return written;
}
```

这部分是将消息写到FileChannel的ByteBuffer中。FileChannel是一个java的NioChannel，可以大幅度提高文件的读写效率。

3. 如果需要，则新增位移索引和时间戳索引项，而添加索引的append方法，实现也很简单

   ```scala
   def append(offset: Long, position: Int): Unit = {
       inLock(lock) {
         if (_entries == 0 || offset > _lastOffset) {
           mmap.putInt(relativeOffset(offset))
           mmap.putInt(position)
           _entries += 1
           _lastOffset = offset
           require(_entries * entrySize == mmap.position(), s"$entries entries but file position in index is ${mmap.position()}.")
         } else {
           ...
         }
       }
     }
   ```

   就是用mmap将相对位移和物理位置写到索引文件。

### 3. Read

```scala
def read(startOffset: Long,
           maxSize: Int,
           maxPosition: Long = size,
           minOneMessage: Boolean = false): FetchDataInfo = {
    if (maxSize < 0)
      throw new IllegalArgumentException(s"Invalid max size $maxSize for log read from segment $log")

    val startOffsetAndSize = translateOffset(startOffset)

    // if the start position is already off the end of the log, return null
    if (startOffsetAndSize == null)
      return null

    val startPosition = startOffsetAndSize.position
    val offsetMetadata = LogOffsetMetadata(startOffset, this.baseOffset, startPosition)

    val adjustedMaxSize =
      if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
      else maxSize

    // return a log segment but with zero size in the case below
    if (adjustedMaxSize == 0)
      return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY)

    // calculate the length of the message set to read based on whether or not they gave us a maxOffset
    val fetchSize: Int = min((maxPosition - startPosition).toInt, adjustedMaxSize)

    FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),
      firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
```



### 4. 页缓存

在操作系统中，文件是以物理页的形式存储在磁盘上的，当一个进程去读取一个文件的内容时，也是按页读取，会读取页目录，然后找到页表，再通过页表将对应的物理页加载到内存中。而操作系统提供了页缓存来提高这个过程的效率。

1. 读取文件时，会首先读取页缓存，如果页缓存中已经存在要读取的页，则无需去磁盘读取，如果页缓存中不存在，则去读取磁盘，并且将物理页加入到页缓存
2. 写文件时，会将文件内容写到页缓存，并定期将页缓存的内容刷到磁盘中。

对kafka来说，日志文件的读写都是非常频繁的，如果将物理页都缓存到进程中，由于java对象本身有额外的开销，会导致占用的堆内存变大，gc的频率变得更加频繁，从而降低kafka的请求相应速度。



因此kafka中大量的使用页缓存来代替内存操作，日志的读写都基于页缓存，大大提高了kafka的吞吐率，与此同时也存在一个问题:kafka的副本会同步leader副本的数据，而某个时刻，页缓存的内容并没有同步到磁盘，此时leader宕机了，会从新选举出来一个leader，而此前页缓存的数据并没有同步到follower副本，新的leader副本就丢失了这部分数据。



考虑到这个问题，kafka提供了实时刷盘机制，可以立即将页缓存的内容同步到磁盘。



页缓存还存在另外一个问题，会直接影响kafka的性能: PageCache竞争



Linux运用LRU算法实现对页缓存的淘汰策略，页缓存的容量是有限的，Linux会将那些很久没有命中的页缓存淘汰掉。当一台机器存在多个Kafka consumer时，如果consumer的消费速度都很快，那producer写到页缓存的数据可以马上被consumer消费掉，不存在PageCache的竞争问题。



但是如果某些consumer消费存在很高的延时，它们来消费时，如果PageCache的空间不足，很可能他们要消费的数据页缓存已经被淘汰了，只能触发磁盘io去磁盘读取，随后Linux会将读取到的数据加到页缓存，由于LRU的机制，其他实时消费的页缓存会被淘汰，造成其他实时消费的consumer被迫去

### 5. 零拷贝



### 6. 二分法查找


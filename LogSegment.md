LogSegment

一个logsegment由以下文件构成:


消息文件 :

位移索引

时间戳索引



一个topic的一个partition对应一个segment



### append方法

```scala
def append(largestOffset: Long,
           largestTimestamp: Long,
           shallowOffsetOfMaxTimestamp: Long,
           records: MemoryRecords): Unit = {
  if (records.sizeInBytes > 0) {
    val physicalPosition = log.sizeInBytes()
    if (physicalPosition == 0)
      rollingBasedTimestamp = Some(largestTimestamp)

    //判断offset是否合法
    ensureOffsetInRange(largestOffset)

    // 将内存中的消息写入到操作系统的页缓存
    val appendedBytes = log.append(records)
    
    if (largestTimestamp > maxTimestampSoFar) {
      maxTimestampSoFar = largestTimestamp
      offsetOfMaxTimestampSoFar = shallowOffsetOfMaxTimestamp
    }
    // append an entry to the index (if needed)
    if (bytesSinceLastIndexEntry > indexIntervalBytes) {
      offsetIndex.append(largestOffset, physicalPosition)
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }
}
```





操作系统mmap


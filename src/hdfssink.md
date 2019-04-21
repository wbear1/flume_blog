# Sink之HDFSEventSink具体实现

1. HDFSEventSink的主流程：
- 1、从channel批量获取Event（这里得考虑transaction）；
- 2、根据你的sink配置来决定Event该写到哪个文件，这时候会拼路径和文件名：realPath和realName，得到文件的全路径：lookupPath=realPath + "/" + realName;
- 3、根据上一步的lookupPath获取BucketWriter对象，调用其append方法写Event；
  - a、每个文件会有一个对应的BucketWriter对象来负责写数据，所以HDFSEventSinks中采用Map<String, BucketWriter> sfWriters来保存文件和BucketWriter的对应关系。
  - b、当文件关闭后，会将相应的BucketWriter从Map移除；
- 4、当取到batchSize数量的Event后，则循环调用BucketWriter对象的flush方法，将数据刷到HDFS上。

下图是简化的流程图：
![flow](https://github.com/wbear1/flume_blog/blob/master/img/hdfssink/flow.png)

具体实现如下（为了便于理解，去除了原代码里的一些判断和异常捕获语句）：
```java
public Status process() throws EventDeliveryException {
  Channel channel = getChannel();
  Transaction transaction = channel.getTransaction();//开启事务
  List<BucketWriter> writers = Lists.newArrayList();
  transaction.begin();//开始事务
  try {
    int txnEventCount = 0;
    for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {//从channel里取batchSize数量的event
      Event event = channel.take();
      if (event == null) {
        break;
      }
 
      // reconstruct the path name by substituting place holders
      String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
          timeZone, needRounding, roundUnit, roundValue, useLocalTime);
      String realName = BucketPath.escapeString(fileName, event.getHeaders(),
        timeZone, needRounding, roundUnit, roundValue, useLocalTime);
 
      String lookupPath = realPath + DIRECTORY_DELIMITER + realName;
      BucketWriter bucketWriter = sfWriters.get(lookupPath);//从缓存里取bucketWriter对象，无则创建
      HDFSWriter hdfsWriter = writerFactory.getWriter(fileType);//根据文件类型创建HDFSWriter对象
      if(bucketWriter == null){
        initializeBucketWriter(realPath, realName, lookupPath, hdfsWriter, closeCallback);
      }
      sfWriters.put(lookupPath, bucketWriter);
 
      if (!writers.contains(bucketWriter)) {
        writers.add(bucketWriter);
      }
 
      bucketWriter.append(event);//将Event写入bucketWriter
    }
 
    // flush all pending buckets before committing the transaction
    for (BucketWriter bucketWriter : writers) {//将所有的bucketWriter里的Event真正flush到HDFS上
      bucketWriter.flush();
    }
 
    transaction.commit();//提交事务
 
    if (txnEventCount < 1) {
      return Status.BACKOFF;
    } else {
      sinkCounter.addToEventDrainSuccessCount(txnEventCount);
      return Status.READY;
    }
  } finally {
    transaction.close();
  }
}
```

2. 详细介绍上面提到的BucketWriter、HDFSWriter

BucketWriter类主要负责hdfs sink的数据写操作，其中有三个主要方法：append、flush和close，顾名思义，不再赘述。

HDFSWriter类是一个接口，负责将数据写到HDFS上，其中主要有4个方法：open、append、sync和close。与BucketWriter的区别：BucketWriter是更上层的类，直接面向的是hdfs sink，不关心具体的文件类型；而HDFSWriter是直接写HDFS文件的，因此HDFSWriter是接口类型，不同的hdfs文件类型会有不同的实现，而BucketWriter最终也是调用HDFSWriter的这几个方法来将数据写到HDFS。HFDSWriter有三个实现类：HDFSDataStream、HDFSSequenceFile和HDFSCompressedDataStream。HDFSDataStream不压缩输出的文件，按原始格式输出；HDFSSequenceFile以二进制的格式输出文件；HDFSCompressedDataStream按指定的压缩格式输出文件。
HDFSWriter接口定义如下：
![HDFSWriter](https://github.com/wbear1/flume_blog/blob/master/img/hdfssink/HDFSWriter.png)

这几个类的之间的调用关系如下，其中FsDataOutputStream是hadoop提供的类。可以看到数据刷到datanode上，是通过FsDataOutputStream的hflush或sync来实现的，具体用哪个跟hadoop的版本有关，两者的区别可参考：http://www.cnblogs.com/foxmailed/p/4145330.html
![output](https://github.com/wbear1/flume_blog/blob/master/img/hdfssink/output.png)

3. HDFSEventSink的主要配置

https://flume.apache.org/releases/content/1.6.0/FlumeUserGuide.html#hdfs-sink

其中罗列几项对性能影响较大的配置：

|Name|Default|Description|
|:-------:|:-------:|:-------:|
|hdfs.batchSize|100|number of events written to file before it is flushed to HDFS   这个配置对sink的输出吞吐量有很大影响，因为上面提到的open和close开销较大|
|hdfs.maxOpenFiles|5000|Allow only this number of open files. If this number is exceeded, the oldest file is closed.  即上面提到的sfWriters的大小，超过就采用LRU算法关闭某些文件|
|hdfs.threadsPoolSize|10|Number of threads per HDFS sink for HDFS IO ops (open, write, etc.)  open、append、flush和close操作都由该线程池来处理|
|hdfs.rollTimerPoolSize|1|Number of threads per HDFS sink for scheduling timed file rolling  文件滚动由该线程池来处理|


下面讲讲文件滚动策略，有5种策略，上面的配置其实没有说清楚：
1）基于时间，配置项是hdfs.rollInterval，默认是30，单位是秒；0表示文件不基于时间滚动。
2）基于文件大小，配置项是hdfs.rollSize，默认是1024，单位是字节；0表示文件不基于大小滚动。
3）基于hdfs文件副本数量，配置项是hdfs.minBlockReplicas，默认读取hadoop上的配置。当前文件副本数量小于hdfs.minBlockReplicas时，文件滚动。1表示文件不基于hdfs文件副本数量滚动。
4）基于event数量，配置项是hdfs.rollCount，默认是10；0表示文件不基于event数量滚动。
5）基于文件闲置时间，表示文件在指定时间内没有数据写入，则文件滚动。配置项是hdfs.idleTimeout，默认是0，表示文件不基于文件闲置时间滚动。

当这几个策略同时存在的情况下，优先级是：基于文件闲置时间 > 基于hdfs文件副本数量 > 基于event数量 > 基于文件大小 > 基于文件时间 

下面这篇博客对这几种策略的实现讲得比较详细，可供参考。http://www.jianshu.com/p/4f43780c82e9
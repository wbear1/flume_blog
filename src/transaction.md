# Transaction设计与实现

channel transaction：对于source而言，要么将批量的event全部发送到channel，要么不发；对于sink而言，从channel批量取event要么成功，要么不取。
可参考博客：http://blog.csdn.net/wsscy2004/article/details/39696095

1. 以MemoryChannel为例，介绍Transaction的实现。
Transaction有四个方法，按一定顺序调用。如下图所示
![methods](https://github.com/wbear1/flume_blog/blob/master/img/transaction/methods.png)
![invoke](https://github.com/wbear1/flume_blog/blob/master/img/transaction/invoke.png)

ch.getTransaction()方法取得当前线程的Transaction对象，没有则新建。Transaction对象与线程绑定，由ThreadLocal存储，在BasicChannelSemantics里有
```java
private ThreadLocal<BasicTransactionSemantics> currentTransaction = new ThreadLocal<BasicTransactionSemantics>();
```

2. MemoryTransaction里有两个临时缓冲区putList和taskList，，都是使用LinkedBlockingDeque类型。putList用于暂存即将放入channel的event，taskList用于暂存刚从channel取出来的event。MemoryChannel使用LinkedBlockingDeque类型的queue保存event，内部主要使用三个Semaphore(queueRemaing、byteRemaing和queueStored)和synchronized(queueLock)来实现事务。如下图所示
![memTrans](https://github.com/wbear1/flume_blog/blob/master/img/transaction/memTrans.png)

3. source向channel发送event，在ChannelProcessor里的processEventBatch调用。分5步进行。

1）首先创建事务：reqChannel.getTransaction
2）开启事务：tx.begin()
3）发送逻辑：reqChannel.put(event)，内部调用了Transaction的doPut方法，即将event放入临时缓冲区putList里
4）提交事务：tx.commit()，内部调用了Transaction的doCommit方法，先检查queue剩余空间是否足够（字节数和event数量），然后将putList里的event移至queue。
5）若出现异常的话，事务回滚：清空putList

```java
Transaction tx = reqChannel.getTransaction();//创建事务
Preconditions.checkNotNull(tx, "Transaction object must not be null");
try {
tx.begin();
 
List<Event> batch = reqChannelQueue.get(reqChannel);
 
for (Event event : batch) {
reqChannel.put(event);//调用channel.put方法，内部调用Transaction的doPut方法
}
 
tx.commit();//提交事务
} catch (Throwable t) {
tx.rollback();//事务回滚
if (t instanceof Error) {
LOG.error("Error while writing to required channel: " +
reqChannel, t);
throw (Error) t;
} else {
throw new ChannelException("Unable to put batch on required " +
"channel: " + reqChannel, t);
}
} finally {
if (tx != null) {
tx.close();
}
}
```

MemoryTransaction的doPut、doCommit和doRollback方法，这三个方法蛮有意思的，值得好好品味一番。

```java
protected void doPut(Event event) throws InterruptedException {
channelCounter.incrementEventPutAttemptCount();
int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);  //计算字节数。这里不是直接算具体的字节数，而是按槽（byteCapacitySlotSize，默认是100字节）来算。
 
if (!putList.offer(event)) {
throw new ChannelException(
"Put queue for MemoryTransaction of capacity " +
putList.size() + " full, consider committing more frequently, " +
"increasing capacity or increasing thread count");
}
putByteCounter += eventByteSize;
}
```

```java
@Override
protected void doCommit() throws InterruptedException {
int remainingChange = takeList.size() - putList.size();
if(remainingChange < 0) {
//保证剩余的字节数能够满足event大小
if(!bytesRemaining.tryAcquire(putByteCounter, keepAlive,
TimeUnit.SECONDS)) {
throw new ChannelException("Cannot commit transaction. Byte capacity " +
"allocated to store event body " + byteCapacity * byteCapacitySlotSize +
"reached. Please increase heap space/byte capacity allocated to " +
"the channel as the sinks may not be keeping up with the sources");
}
//保证剩余的可放入event数量大于即将放入的event数量
if(!queueRemaining.tryAcquire(-remainingChange, keepAlive, TimeUnit.SECONDS)) {
bytesRemaining.release(putByteCounter);
throw new ChannelFullException("Space for commit to queue couldn't be acquired." +
" Sinks are likely not keeping up with sources, or the buffer size is too tight");
}
}
int puts = putList.size();
int takes = takeList.size();
synchronized(queueLock) {
if(puts > 0 ) {
//将临时缓冲区putList里的event移至queue
while(!putList.isEmpty()) {
if(!queue.offer(putList.removeFirst())) {
throw new RuntimeException("Queue add failed, this shouldn't be able to happen");
}
}
}
putList.clear();
takeList.clear();
}
bytesRemaining.release(takeByteCounter);//将event移除后，可用空间又变大了
takeByteCounter = 0;
putByteCounter = 0;
 
queueStored.release(puts);//已用空间变大
if(remainingChange > 0) {
queueRemaining.release(remainingChange);
}
if (puts > 0) {
channelCounter.addToEventPutSuccessCount(puts);//用于统计用，可以忽略
}
if (takes > 0) {
channelCounter.addToEventTakeSuccessCount(takes);//用于统计用，可以忽略
}
 
channelCounter.setChannelSize(queue.size());
}
```

```java
@Override
protected void doRollback() {
int takes = takeList.size();
synchronized(queueLock) {
Preconditions.checkState(queue.remainingCapacity() >= takeList.size(), "Not enough space in memory channel " +
"queue to rollback takes. This should never happen, please report");
while(!takeList.isEmpty()) {
queue.addFirst(takeList.removeLast());
}
putList.clear();
}
bytesRemaining.release(putByteCounter);
putByteCounter = 0;
takeByteCounter = 0;
 
queueStored.release(takes);
channelCounter.setChannelSize(queue.size());
}
```

注意：在MemoryTransaction的设计里，tx.begin()和tx.commit()之间的逻辑除了单独put和单独take，也可以同时有put和take操作（不过，暂时没想到什么场景。。。），所以在doCommit和doRollback方法里，同时考虑putList和takeList。

4. sink从channel里取event的流程类似，将doPut换作doTake。


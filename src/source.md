# Source设计与实现

Flume source的工作流程如下图所示：
![flow](https://github.com/wbear1/flume_blog/blob/master/img/source/flow.png)
![flow2](https://github.com/wbear1/flume_blog/blob/master/img/source/flow2.png)

具体解释下：
1）source批量取event，比如从数据库里读取100行记录；
2）对这批event进行处理，比如过滤某些不符合要求的event，或者给event打上某些标识，表示发往某个channel；
3）选择channel，一个source的event可以发往多个channel；
4）将event发送到channel的队列里，这里要考虑transaction，关于transaction另作介绍；
5）以上4步抛出异常或者没有取到数据，均为返回BACKOFF状态，这时候source就sleep一段时间然后再重复该过程。否则则继续。
具体调用关系如下：
![invoke](https://github.com/wbear1/flume_blog/blob/master/img/source/invoke.png)

下面再详细介绍ChannelSelector、Interceptor和ChannelProcessor。

1. ChannelSelector
flume提供了两种channel selector的类型：replicating和multiplexing。如下图：
![selector](https://github.com/wbear1/flume_blog/blob/master/img/source/selector.png)

replicating表示一个event发往每个channel，multiplexing表示一个event根据相应的规则发往指定的某些channel。默认类型是replicating。
![replicating](https://github.com/wbear1/flume_blog/blob/master/img/source/replicating.png)

上面source1的event会发往channel1和channel2
![replicating2](https://github.com/wbear1/flume_blog/blob/master/img/source/replicating2.png)

上面source1会根据someHeader字段来决定event发往哪个channel，如果someHeader的值为Value1，则发往channel1；如果someHeader的值为Value3，则发往channel2；如果someHeader的值为Value2，则发往channel1和channel2；如果someHeader为其它值，则发往channel2。

selector还可以配置optional，表示该selector不保证事务，event发送失败，则忽略不重试。selector默认都是非optional的，即required类型。
![optional](https://github.com/wbear1/flume_blog/blob/master/img/source/optional.png)

上面avro-AppSrv-source1如果event的header里的State值为CA的话，则发送到mem-channel-1和file-channel-2，若失败，也不重试；如果State的值为CA的话，event发送到mem-channel-1，失败则会重试。

2. Interceptor
- 多个Interceptor组成一个InterceptorChain，Event流过时依次执行Interceptor的intercept方法。下面以HostInterceptor简单介绍其实现。
- Interceptor有四个方法和一个内部类Builder。initialize()用于初始化Interceptor，intercept(Event)和intercept(List<Event)分别用于处理单个Event和处理批量Event，如上面所示HostInterceptor的intercept(Event)就是将host信息塞到Event的headers里，close()用于Interceptor的收尾工作。内部类Builder只有一个方法build()，用于实例化相应的Interceptor。比如HostInterceptor实例的Builder，就是用于实例化HostInterceptor。
当Interceptor需要丢弃Event请注意：intercept(Event)方法直接返回null即可；intercept(List<Event>)方法将要丢弃的Event不加入到返回的list，如果要丢弃所有的Event，则返回一个空的LinkedList对象，不是nul。

3. ChannelProcessor

ChannelProcessor负责将source的event，发送到相应的channel里，包括required channels和optional channels。看里面的主要方法processEventBatch(List<Event>),source在发送数据时会调用该方法。
```java
public void processEventBatch(List<Event> events) {
  Preconditions.checkNotNull(events, "Event list must not be null");
 
  //1、先经过 interceptorChain的处理
  events = interceptorChain.intercept(events);
 
  Map<Channel, List<Event>> reqChannelQueue =
      new LinkedHashMap<Channel, List<Event>>();
 
  Map<Channel, List<Event>> optChannelQueue =
      new LinkedHashMap<Channel, List<Event>>();
 
  //2、将event放到上面的reqChannelQueue和optChannelQueue里，准备发送到reqChannel和optChannel。
  //这段代码可以优化，没必要遍历events
  for (Event event : events) {
    List<Channel> reqChannels = selector.getRequiredChannels(event);
 
    for (Channel ch : reqChannels) {
      List<Event> eventQueue = reqChannelQueue.get(ch);
      if (eventQueue == null) {
        eventQueue = new ArrayList<Event>();
        reqChannelQueue.put(ch, eventQueue);
      }
      eventQueue.add(event);
    }
 
    List<Channel> optChannels = selector.getOptionalChannels(event);
 
    for (Channel ch: optChannels) {
      List<Event> eventQueue = optChannelQueue.get(ch);
      if (eventQueue == null) {
        eventQueue = new ArrayList<Event>();
        optChannelQueue.put(ch, eventQueue);
      }
 
      eventQueue.add(event);
    }
  }
 
  // 3、调用相应reqChannel的put方法，将event塞到reqChannel
  for (Channel reqChannel : reqChannelQueue.keySet()) {
    Transaction tx = reqChannel.getTransaction();//创建事务
    Preconditions.checkNotNull(tx, "Transaction object must not be null");
    try {
      tx.begin();
 
      List<Event> batch = reqChannelQueue.get(reqChannel);
 
      for (Event event : batch) {
        reqChannel.put(event);//发送Event
      }
 
      tx.commit();//提交事务
    } catch (Throwable t) {
      tx.rollback();//事务回滚，Error和Exception都会回滚。
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
  }
 
  // 4、调用相应optChannel的put方法，将event塞到optChannel，和步骤3基本一样，除了回滚逻辑不一样外。
  for (Channel optChannel : optChannelQueue.keySet()) {
    Transaction tx = optChannel.getTransaction();
    Preconditions.checkNotNull(tx, "Transaction object must not be null");
    try {
      tx.begin();
 
      List<Event> batch = optChannelQueue.get(optChannel);
 
      for (Event event : batch ) {
        optChannel.put(event);
      }
 
      tx.commit();
    } catch (Throwable t) {
      tx.rollback();//事务回滚，只有出现Error时，才会回滚；出现Exception，忽略。
      LOG.error("Unable to put batch on optional channel: " + optChannel, t);
      if (t instanceof Error) {
        throw (Error) t;
      }
    } finally {
      if (tx != null) {
        tx.close();
      }
    }
  }
}
```
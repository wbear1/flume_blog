# 初识之Application

Apache Flume is a distributed, reliable, and available system for efficiently collecting, aggregating and moving large amounts of log data from many different sources to a centralized data store.

用下图简单解释下
![component](https://github.com/wbear1/flume_blog/blob/master/img/intro/component.png)

Flume就是负责将存储在源（A）的数据转移至目标（B）。Flume里主要有三个主要的组件：Source、Channel和Sink，Source负责从源A取数据，然后将数据暂在channel，而Sink则从Channel里取数据后发送到目标B。数据在整个传输过程中的最小单位是一个event，即源A的一条记录，比如数据库表里的一行记录、Kafka的一个record。
 Flume中主要有如下组件：ConfigurationProvider、SourceRunner、Source、SinkRunner、Sink、Channel。ConfigurationProvider用于读取配置；SourceRunner中会启动一个线程，调用Source的process来获取event然后发送到channel；SinkRunner中会启动一个线程，调用Sink的process方法来从channel读取event并负责发送到目标；而Channel中维护一个队列，用来暂存source发送的数据。这些组件采用有限状态机的方式来管理。
 如下图所示：
![stat](https://github.com/wbear1/flume_blog/blob/master/img/intro/stat.png)

初始状态都是IDLE，当状态为START时，则会调用组件的start方法；当状态为STOP时，则调用组件的stop方法；当状态为ERROR，则打印错误信息。
Flume启动之后，即执行Application的main方法，流程如下图所示：
![flow](https://github.com/wbear1/flume_blog/blob/master/img/intro/flow.png)

supervisor的线程池会监控各个组件的状态，根据组件的状态执行各自的对应方法。同时，如果配置文件修改，则会通知Application停止当前的组件，启动新加载的组件。
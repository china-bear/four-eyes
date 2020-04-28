
 

#### 1、本地启动flink报错“Caused by: java.lang.NoClassDefFoundError:org/apache/flink/streaming/api/functions/source/SourceFunction”
  将flink-java和flink-streaming-java_${scala.binary.version}这两个依赖由provided改为compile
如![在这里插入图片描述](https://img-blog.csdnimg.cn/20200416210214483.png)
  参考：[https://stackoverflow.com/questions/54106187/apache-flink-java-lang-noclassdeffounderror](https://stackoverflow.com/questions/54106187/apache-flink-java-lang-noclassdeffounderror)
  
#### 2、实时去重
  遇到一个业务需求，将订单日志left join多张维表后写入实时数仓，因为同一笔订单会有多个状态，故会有多条原始日志，需要将全站的订单号进行实时去重，source是sls，sink是kafka，但是发现kafka中出现订单重复的问题。问题的原因是因为代码中进行了多次left join，所以需要在写入sink之前最后再进行一次distinct。实时去重一般来说有三种方案。
1、使用MapState(继承RichFunction)，可以自定义，好处是方便，但是会导致State一直增长。（当前flink集群的checkpoint还是基于HDFS的，所以不能使用RocketDB ）
2、布隆过滤器，精度有缺少，直接不考虑
3、引入第三方外部K-V存储，如Redis、HBase
我们考虑到需求上线时间，选择了第一种～

####  3、本地运行OK，但是打包到flink集群上出现问题：NoSuchMethodError
  jar在flink集群提交阶段就报错：“Caused by: java.lang.NoSuchMethodError: okhttp3.HttpUrl.get(Ljava/lang/String;)Lokhttp3/HttpUrl;”
一般出现NoSuchMethodError会有两种可能，
第一、提交节点确实少包，比如jar中需要HBase、ES这些第三方依赖而提交节点依赖包不完整时，这时只能一个一个包慢慢上传到提交节点上了，或者干脆拷贝个对应的client的lib(有jar冲突问题)。正常来说一个集群会固定一个服务器作为提交节点，依赖也会最全。
第二、jar包冲突。本次就是这个问题。排查后发现是'com.squareup.okhttp3'jar中类HttpUrl提供这个方法，本地版本是3.14.4，是没有问题的。于是我把该jar的pom注释了，在代码中还能搜索到HttpUrl～终于发现问题，influxdb-java 中引入了3.8.1的版本。整体过程
1、先全局搜索这个类，无果后注释显示的pom的文件
2、在pom上进行Diagrams -> show dependencies -> ctrl+f ，或者类依赖太多的时候可以 mvn dependency:tree > depend.log 导出文件
3、直接界面exclude或在pom中加exclusion

#### 4、kafka source 消息‘假堆积’
   今天和‘同事’遇到一个非常有趣的问题，在Kafka监控页面上可以看到该topic的消息会堆积到10万条左右，然后再很快的消费完毕归为0，然后又开始堆积到10万条左右，时间间隔都是3分钟。第一反应，反压？ 但是反压的也太整齐了吧。看了一眼代码，哈哈，CheckPoint的时间间隔是3分钟，而且是Exactly_Once的。
    所以这3分钟的延迟是必须付出的代价了，算不上问题

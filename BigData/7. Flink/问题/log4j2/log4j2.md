@[toc]
### 1. 背景
很多同学在进行Flink开发时，无论是使用log4j或log4j2，常常出现各种问题，如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613221650354.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExMjQwNDY2MTk2,size_16,color_FFFFFF,t_70#pic_center)今天我们就要拨开云雾见天日，聊聊日志相关的知识，搞清楚这些报错的原因。

众所周知，现代框架都是用门面模式进行日志输出，例如使用Slf4j中的接口输出日志，具体实现类需要由log4j，log4j2，logback等日志框架进行实现，如Flink的类中是这样输出日志的：

```java
// org.apache.flink.api.java.ClosureCleaner

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
@Internal
public class ClosureCleaner {
	private static final Logger LOG = LoggerFactory.getLogger(ClosureCleaner.class);
...
}
```
这种设计使得用户可以自由地进行日志框架的切换。

### 2. 日志门面slf4j
slf4j全名Simple Logging Facade for Java，为java提供的简单日志Facade。Facade门面说白了就是接口。它允许用户以自己的喜好，在工程中通过slf4j接入不同的日志系统。slf4j入口就是众多接口的集合，它不负责具体的日志实现，只在编译时负责寻找合适的日志系统进行绑定。具体有哪些接口，全部都定义在slf4j-api中。查看slf4j-api源码就可以发现，里面除了`public final class LoggerFactory`类之外，都是接口定义。因此slf4j-api本质就是一个接口定义。要想使用slf4j日志门面，需要引入以下依赖

```xml
 <dependency>
	<groupId>org.slf4j</groupId>
	<artifactId>slf4j-api</artifactId>
	<version>1.7.25</version>
</dependency>
```

这个包只有日志的接口，并没有实现，所以如果要使用就得再给它提供一个实现了些接口的日志框架包，比如：log4j，log4j2，logback等日志框架包，但是这些日志实现又不能通过接口直接调用，实现上他们根本就和slf4j-api不一致，因此slf4j和日志框架之间又增加了一层桥接器来转换各日志实现包的使用，比如slf4j-log4j12，log4j-slf4j-impl等。

#### 2.1 slf4j + log4j
Log4j + Slf4j的使用组合最为常见，依赖关系如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613215609174.png#pic_center)
使用时要引入maven依赖：

```xml
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.25</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
```
#### 2.2 slf4j + log4j2
但是Log4j目前已经停止更新了。Apache推出了新的Log4j2来代替Log4j，Log4j2是对Log4j的升级，与其前身Log4j相比有了显着的改进，并提供了许多Logback可用的改进，同时解决了Logback体系结构中的一些固有问题。因此，Log4j2 + Slf4j应该是未来的大势所趋。其依赖关系如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613220440479.png#pic_center)
使用时要引入maven依赖：

```xml
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>2.9.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.9.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.9.1</version>
        </dependency>
```
无论是使用log4j还是log4j2，尤其要记得引入各自的桥接器，否在就会报文章开头找不到配置的错误！
### 3. 避免冲突
由于log4j2的性能优异且是大势所趋，本文决定使用log4j2作为Flink开发的日志框架，一切按照上文配置就绪，却在运行时抛出如下错误：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613222742399.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExMjQwNDY2MTk2,size_16,color_FFFFFF,t_70#pic_center)
如报错所示，在classpath下发现了两个桥接器，很明显，有某个依赖间接依赖了log4j的桥接器`slf4j-log4j12`，使得classpath下同时存在了`slf4j-log4j12`和`log4j-slf4j-impl`这两个桥接器，经过`mvn dependency:tree -Dincludes=:slf4j-log4j12`命令分析，我们找出了这个依赖，在其中将`slf4j-log4j12`进行了排除：

```xml
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-redis_2.11</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
            <version>1.1.5</version>
        </dependency>
```
错误顺利消失，正当喜悦之时，有报出如下错误：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613223250295.jpg#pic_center)
乍一看有些懵，不过仔细想想，刚刚排除了桥接器的冲突，这个问题是否是因为项目间接依赖了log4j，而导致底层日志框架log4j和log4j2产生了冲突呢？日志门面slf4j虽然正确找到了log4j2的桥接器`log4j-slf4j-impl`，但是接下来会不会又根据maven的短路优先原则，歪打正着地找到了log4j框架（`log4j-slf4j-impl`桥接器本应去寻找log4j2框架）？本文意图使用log4j2，因此配置了log4j2.xml，自然不会去配置log4j.properties，更无法找到log4j.properties中的appender等组件了。

果不其然，在找到并排除了间接依赖的log4j之后，错误也随之消失：

```xml
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-redis_2.11</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
            <version>1.1.5</version>
        </dependency>
```

### 4. 总结
1. 不管使用log4j还是log4j2，别忘了在pom.xml中配置桥接器！
2. 使用一个日志框架时，请排除另一个日志框架所使用的一切直接和间接依赖，避免一些令人迷惑的报错！
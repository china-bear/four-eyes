[TOC]

### 1. 为什么要进行插件化开发？

1、解决依赖冲突

设想如下场景，不同Hadoop厂商例如HDP和CDH的中使用了`hadoop-yarn-client`

的不同版本，如果项目同时依赖HDP和CDH，都通过maven进行依赖引入。此时maven会根据`最短路径原则`和`优先声明`原则进行`hadoop-yarn-client`版本的选择，但此时HDP和CDH依赖都是通过AppClassLoader进行加载，其对应类加载器命名空间中只能存在一个`hadoop-yarn-client`版本，导致HDP和CDH必然只能使用同一个版本的`hadoop-yarn-client`，这就产生了依赖冲突问题。因此，我们的需求是同时引入两个版本的`hadoop-yarn-client`，根据项目需要，动态地加载不同版本的`hadoop-yarn-client`。此时，插件化开发中的类加载器命名空间技术便应运而生，它将不同版本但全限定名相同的两个类，加载到不同的命名空间进行隔离。我们只需要根据项目需要，动态引用不同命名空间的类即可。

2、动态加载

当我们生产上的项目不能轻易停止或重启时，我们又想临时接入Fusioninsight集群，此时项目就要动态地加载Fusioninsight所使用的又一个版本的`hadoop-yarn-client`。SPI和文件监听技术粉墨登场，SPI技术有点类似IOC的思想，它将类加载的控制权转移到程序之外，通过读取配置文件中关于实现类的描述，来加载接口的具体实现类。文件监听技术则可以动态地监听JAR包的新增与更新，在发生变化时，再次将更新或新增后的插件加载到JVM中。

综上所述，插件化开发可以实现避免依赖冲突、运⾏时通过构建参数动态加载插件等功能。本章将进行插件化中一些关键技术的解释，并进行一些小实验假以佐证。



### 2.类加载器命名空间

假设类加载器A有一个父类加载器B，B有一个父类加载器C，用A去加载class，根据双亲委派原则，最终用C去加载了class且加载成功，则A和B叫做class的初始类加载器，C叫做class的定义类加载器。换句话说，A委托B，B又委托C最终加载了class，则最终加载class的类加载器C叫做class的定义类加载器，A和B叫做class的初始类加载器。

每个类加载器都维护了一个列表，如果一个class将该类加载器标记为定义类加载器或者初始类加载器，则该class会存入该类加载器的列表中，列表即命名空间。

class会在被其标记为初始类加载器和定义类加载器的类加载器命名空间中共享，用上面的例子讲，class x经过A、B、C三个类加载器进行双亲委派，并最终由C进行加载，则class x会在三者的命名空间中共享，即虽然class x是由C进行加载，但是A和B命名空间中的其他class都可以引用到class x。



### 3. SPI

SPI全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来加载和替换第三方插件。在面向对象的设计里，一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了开闭原则，Java SPI就是为某个接口寻找服务实现的机制，Java Spi的核心思想就是**解耦**。

#### 3.1 示例

我们以一个例子展开，进行SPI源码的讲解：

1. 支付接口

```java
package com.imooc.spi;

import java.math.BigDecimal;

public interface PayService {

    void pay(BigDecimal price);
}
```

2. 第三方支付实现类

```java
package com.imooc.spi;

import java.math.BigDecimal;
// 支付宝支付
public class AlipayService implements PayService{

    public void pay(BigDecimal price) {
        System.out.println("使用支付宝支付");
    }
}
```

```java
package com.imooc.spi;

import java.math.BigDecimal;
// 微信支付 
public class WechatPayService implements PayService{

    public void pay(BigDecimal price) {
        System.out.println("使用微信支付");
    }
}

```

3. resources目录下创建目录META-INF/services/com.imooc.spi.PayService文件，文件内容为实现类的全限定名

```properties
com.imooc.spi.AlipayService
com.imooc.spi.WechatPayService
```

4. 创建测试类

```java
package com.imooc.spi;

import com.util.ServiceLoader;

import java.math.BigDecimal;

public class PayTests {

    public static void main(String[] args) {
        ServiceLoader<PayService> payServices = ServiceLoader.load(PayService.class);
        for (PayService payService : payServices) {
            payService.pay(new BigDecimal(1));
        }

      	// 通过迭代器进行遍历
        /*
        Iterator<SPIService> spiServiceIterator = serviceLoaders.iterator();
        while (spiServiceIterator != null && spiServiceIterator.hasNext()) {
            SPIService spiService = spiServiceIterator.next();
            System.out.println(spiService.getClass().getName() + " : " + spiService.pay());
        }
        */
    }
}
//结果：
//使用支付宝支付
//使用微信支付
```

#### 3.2 源码解析

1. load()

```java
public final class ServiceLoader<S> implements Iterable<S> {

    // SPI文件路径的前缀
    private static final String PREFIX = "META-INF/services/";
    // 需要加载的服务的类或接口
    private Class<S> service;
    // 用于定位、加载和实例化提供程序的类加载器
    private ClassLoader loader;
    // 创建ServiceLoader时获取的访问控制上下文
    private final AccessControlContext acc;
    // 按实例化顺序缓存Provider
    private LinkedHashMap<String, S> providers = new LinkedHashMap();
    // 懒加载迭代器
    private LazyIterator lookupIterator;
    
    // 1. 获取上下文类加载器
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }

    // 2. 调用构造方法
    public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
        return new ServiceLoader<>(service, loader);
    }

    // 3. 校验参数和ClassLoad
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }

    //4. 清理缓存容器，实例懒加载迭代器
    public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
    }
}
```

可以看到，load方法仅仅是给ServiceLoader的成员属性service和loader进行了赋值，其中service是接口的class对象，loader是线程上下文类加载器默认为ApplicationClassLoader，最终调用了reload方法将service和loader赋值给了内部类ServiceLoader.LazyIterator的成员属性。



2. hasNext() & next()

众所周知，当对Iterable类使用foreach循环时，其实是在调用其内部迭代器Iterator的hasNext方法和next方法，而ServiceLoader中Iterator类型的匿名内部类中的hasNext方法和next方法又调用了内部类ServiceLoader.LazyIterator中的hasNext方法和next方法，因此ServiceLoader.LazyIterator类的hasNext和next方法会在foreach循环中进行调用。

```java
public final class ServiceLoader<S> implements Iterable<S> {
    // SPI文件路径的前缀
    private static final String PREFIX = "META-INF/services/";
    // 需要加载的服务的类或接口
    private Class<S> service;
    // 用于定位、加载和实例化提供程序的类加载器
    private ClassLoader loader;
    // 创建ServiceLoader时获取的访问控制上下文
    private final AccessControlContext acc;
    // 按实例化顺序缓存Provider
    private LinkedHashMap<String, S> providers = new LinkedHashMap();
    // 懒加载迭代器
    private LazyIterator lookupIterator;

    // 内部迭代器
    public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String, S>> knownProviders
                    = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }

            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    // Private inner class implementing fully-lazy provider lookup
    private class LazyIterator implements Iterator<S> {

        // 需要加载的服务的类或接口
        Class<S> service;
        // 用于定位、加载和实例化提供程序的类加载器
        ClassLoader loader;
        // 枚举类型的资源路径
        Enumeration<URL> configs = null;
        // 迭代器
        Iterator<String> pending = null;
        // 配置文件中下一行className
        String nextName = null;

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }

        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
          //到classpath下寻找以接口全限定名命名的文件，并返回其中的实现类的全限定名
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            // 获取实现类全限定名
            nextName = pending.next();
            return true;
        }

        private S nextService() {
            if (!hasNextService()) throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                // 加载实现类，如果已存在于JVM则直接返回
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service, "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service, "Provider " + cn + " not a subtype");
            }
            try {
                // 实例化并进行类转换
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service, "Provider " + cn + " could not be instantiated", x);
            }
            throw new Error();          // This cannot happen
        }

        public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() {
                        return hasNextService();
                    }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() {
                        return nextService();
                    }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }
    }
}
```

可以看到，hasNextService()方法负责到配置文件中查找当前实现类的全限定名；nextService()方法则负责调用Class.forName方法进行当前实现类的class的返回。



### 4. class缓存查找机制

在分析过SPI的源码之后，我们发现，最终插件会通过`Class.forName(cn, false, loader)`进行加载。但众所周知，如果`Class.forName`发现JVM中已经存在该class，则会直接返回class对象，而不必重复加载。另，如果JVM中不存在该class，则会通过loader类加载器进行加载，而根据《插件化开发基础篇-ClassLoader》中的ClassLoader源码分析可知，loader类加载器中的`loadclass`方法中，会调用`findLoadedClass`方法进行缓存的查找，如果JVM中已经存在该class则直接返回，否则再次进行加载。

要想彻底理顺插件化开发的机制，我们必须搞清楚`Class.forName`和`findLoadedClass`两个方法查找JVM中class的机制，究竟是查找类加载器命名空间下是否存在该class，还是跨命名空间进行JVM全局查找，亦或是其他？我们通过两个实验来寻找答案。

#### 4.1 findLoadedClass缓存查找

首先，自定义一个类加载器`MyClassLoader`

```java
import java.io.*;

public class MyClassLoader extends ClassLoader {

    /**
     * 重写findClass方法
     * @param name 是我们这个类的全路径
     * @return
     * @throws ClassNotFoundException
     */
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {

        try {
            String fileName = name.substring(name.lastIndexOf(".")+1)+".class";
            InputStream is = this.getClass().getResourceAsStream(fileName);
            byte[] b = new byte[is.available()];
            is.read(b);
            return defineClass(name, b, 0, b.length);
        }catch (IOException e){
            throw new ClassNotFoundException(name);
        }
    }
}
```

接下来，进行如下实验：

```java
/**
 * 注意事项：使用如上自定义类加载，且把实验所用到的RemoteADSServiceImpl.class文件放到resources目录下
 */
public class Test {
    public static void main(String[] args) throws Exception {
        java.lang.reflect.Method m = ClassLoader.class.getDeclaredMethod("findLoadedClass", new Class[]{String.class});
        m.setAccessible(true);
        //系统类加载器，AppClassLoader
        ClassLoader cl1 = ClassLoader.getSystemClassLoader();
        //自定义来加载器，没有自己的构造器，自动调用super()，因而父加载器为AppClassLoader
        ClassLoader cl2 = new MyClassLoader();

      
        cl1.loadClass("com.custom.RemoteADSServiceImpl");
        Object test1 = m.invoke(cl1, "com.custom.RemoteADSServiceImpl");
        System.out.println(test1 != null);//true

      
        cl2.loadClass("com.custom.RemoteADSServiceImpl");
        Object test2 = m.invoke(cl2, "com.custom.RemoteADSServiceImpl");
        System.out.println(test2 != null);//false，原因：虽然调用了cl2的加载方法，但是根据双亲委派原则，实际加载使用的是cl1

      
        cl2.loadClass("com.custom.RemoteADSServiceImpl");
        Object test3 = m.invoke(cl1, "com.custom.RemoteADSServiceImpl");
        System.out.println(test3 != null);//true，原因：同test2
    }
}
```

**结论：**findLoadedClass判断class是否存在，与类加载器命名空间无关，它只能返回以当前类加载器作为定义类加载器的class，以当前类加载器作为初始加载器的class则不能被当做缓存进行返回



#### 4.2 Class.forName缓存查找

```java
/**
 * 说明：实验通过在ClassLoader类中的loadclass方法上打断点，判断是否进行类加载
 *      走loadclass即没有命中缓存，使用参数中的classLoader中的loadclass方法重新加载
 *      不走loadclass，即命中了缓存
 */
public class Test {
    public static void main(String[] args) throws Exception {
        java.lang.reflect.Method m = ClassLoader.class.getDeclaredMethod("findLoadedClass", new Class[]{String.class});
        m.setAccessible(true);
        //系统类加载器，AppClassLoader
        ClassLoader cl1 = ClassLoader.getSystemClassLoader();
        //自定义类加载器，没有自己的构造器，自动调用super()，因而父加载器为AppClassLoader
        ClassLoader cl2 = new MyClassLoader();


        Class x = Class.forName("com.custom.RemoteADSServiceImpl", false, cl1);//走loadclass
        Class y = Class.forName("com.custom.RemoteADSServiceImpl", false, cl1);//不走loadclass


        Class x = Class.forName("com.custom.RemoteADSServiceImpl", false, cl1);//走loadclass
        Class y = Class.forName("com.custom.RemoteADSServiceImpl", false, cl2);//走loadclass
//        原因：cl2是cl1的子类加载器，第一步直接用cl1进行加载，cl2并没有被标记为该class的初始加载器，即RemoteADSServiceImpl.class不在cl2的命名空间


        Class x = Class.forName("com.custom.RemoteADSServiceImpl", false, cl2);//走loadclass
        Class y = Class.forName("com.custom.RemoteADSServiceImpl", false, cl1);//不走loadclass
//        原因：cl2是cl1的子类，第一步其实在用cl1进行加载，cl1被RemoteADSServiceImpl.class标记为定义加载器，cl2被RemoteADSServiceImpl.class标记为初始加载器，因此RemoteADSServiceImpl.class在cl1和cl2的命名空间中共享



        Class x = Class.forName("com.custom.RemoteADSServiceImpl", false, cl2);//走loadclass
        Class y = Class.forName("com.custom.RemoteADSServiceImpl", false, cl2);//不走loadclass
//        原因：cl2是cl1的子类，第一步虽然是在用cl1进行加载，但是cl2被RemoteADSServiceImpl.class标记为初始加载器，因此RemoteADSServiceImpl.class在cl1和cl2的命名空间中共享
    }
}
```

**结论：**Class.forName判断class是否存在，与类加载器命名空间相关，如果class将当前类加载器标记为定义类加载器或者初始类加载器，则能直接返回



### 5. 文件监听机制

#### 5.1 示例

```java
public class Test {
    public static void main(String[] args) throws Exception {
        File directory = new File("/Users/djg/Downloads");
        // 轮询间隔 5 秒
        long interval = TimeUnit.SECONDS.toMillis(5);
        // step1:创建observer
        FileAlterationObserver observer = new FileAlterationObserver(directory);
        // step2:设置listener
        observer.addListener(new MyFileListener());
      	// step3:创建monitor
        FileAlterationMonitor monitor = new FileAlterationMonitor(interval, observer);
        // step4:启动monitor
        monitor.start();
    }
}

final class MyFileListener extends FileAlterationListenerAdaptor {
    
    @Override
    public void onDirectoryCreate(File file) {
        System.out.println(file.getName() + " director created.");
    }

    @Override
    public void onDirectoryChange(File file) {
        System.out.println(file.getName() + " director changed.");
    }

    @Override
    public void onDirectoryDelete(File file) {
        System.out.println(file.getName() + " director deleted.");
    }

    @Override
    public void onFileCreate(File file) {
        System.out.println(file.getName() + " created.");
    }

    @Override
    public void onFileChange(File file) {
        System.out.println(file.getName() + " changed.");
    }

    @Override
    public void onFileDelete(File file) {
        System.out.println(file.getName() + " deleted.");
    }
}
```

#### 5.2 源码解析

1. 启动Monitor轮询线程

```java
public final class FileAlterationMonitor implements Runnable {
    private final long interval;
    private final List<FileAlterationObserver> observers = new CopyOnWriteArrayList<FileAlterationObserver>();
    private Thread thread = null;
    private ThreadFactory threadFactory;
    private volatile boolean running = false;
    // 启动方法
    public synchronized void start() throws Exception {
        if (running) {
            throw new IllegalStateException("Monitor is already running");
        }
        for (FileAlterationObserver observer : observers) {
            observer.initialize();
        }
        running = true;
        // 起一个文件监控线程
        if (threadFactory != null) {
            thread = threadFactory.newThread(this);
        } else {
            thread = new Thread(this);
        }
        thread.start();
    }
    public void run() {
        while (running) {
            for (FileAlterationObserver observer : observers) {
                // 每次轮询，执行一次文件监控逻辑
                observer.checkAndNotify();
            }
            if (!running) {
                break;
            }
            // 每隔interval时长轮询一次
            try {
                Thread.sleep(interval);
            } catch (final InterruptedException ignored) {
            }
        }
    }
}
```

2. 文件增删改判断

```java
public class FileAlterationObserver implements Serializable {

    private final List<FileAlterationListener> listeners = new CopyOnWriteArrayList<FileAlterationListener>();
    private final FileEntry rootEntry;
    private final FileFilter fileFilter;
    private final Comparator<File> comparator;

    public void checkAndNotify() {
        // 每次轮询都会执行，FileAlterationListener#onStart一般不需要去复写
        for (FileAlterationListener listener : listeners) {
            listener.onStart(this);
        }

        File rootFile = rootEntry.getFile();
        if (rootFile.exists()) {
            checkAndNotify(rootEntry, rootEntry.getChildren(), listFiles(rootFile));
        } else if (rootEntry.isExists()) {
            checkAndNotify(rootEntry, rootEntry.getChildren(), FileUtils.EMPTY_FILE_ARRAY);
        } else {
            // Didn't exist and still doesn't
        }

        // 每次轮询都会执行，FileAlterationListener#onStop一般不需要去复写
        for (FileAlterationListener listener : listeners) {
            listener.onStop(this);
        }
    }

    // Compare two file lists for files which have been created, modified or deleted
    // previous表示上一次轮询时根目录下的所有文件
    // files表示当前根目录下所有文件
    private void checkAndNotify(FileEntry parent, FileEntry[] previous, File[] files) {
        int c = 0;
        FileEntry[] current = files.length > 0 ? new FileEntry[files.length] : FileEntry.EMPTY_ENTRIES;
        for (FileEntry entry : previous) {
            while (c < files.length && comparator.compare(entry.getFile(), files[c]) > 0) {
                current[c] = createFileEntry(parent, files[c]);
                doCreate(current[c]);
                c++;
            }
            if (c < files.length && comparator.compare(entry.getFile(), files[c]) == 0) {
                // 当文件没有发生增减，判断文件是否被修改
                doMatch(entry, files[c]);
                checkAndNotify(entry, entry.getChildren(), listFiles(files[c]));
                current[c] = entry;
                c++;
            } else {
                checkAndNotify(entry, entry.getChildren(), FileUtils.EMPTY_FILE_ARRAY);
                doDelete(entry);
            }
        }
        for (; c < files.length; c++) {
            current[c] = createFileEntry(parent, files[c]);
            doCreate(current[c]);
        }
        parent.setChildren(current);
    }
    // 判断文件是否发生修改
    private void doMatch(FileEntry entry, File file) {
        if (entry.refresh(file)) {
            for (FileAlterationListener listener : listeners) {
                if (entry.isDirectory()) {
                    listener.onDirectoryChange(file);
                } else {
                    listener.onFileChange(file);
                }
            }
        }
    }
}
```

3. 文件修改具体判断逻辑

```java
public class FileEntry implements Serializable {
    // 将当前文件各项属性和缓存进行对比，FileEntry对象是文件缓存，file参数是当前文件
  	// 取一些关键属性进行对比，如字节长度，最近修改时间，文件名等等
    public boolean refresh(File file) {

        // cache original values
        boolean origExists = exists;
        long origLastModified = lastModified;
        boolean origDirectory = directory;
        long origLength = length;

        // refresh the values
        name = file.getName();
        exists = file.exists();
        directory = exists ? file.isDirectory() : false;
        lastModified = exists ? file.lastModified() : 0;
        length = exists && !directory ? file.length() : 0;

        // Return if there are changes
        return exists != origExists ||
                lastModified != origLastModified ||
                directory != origDirectory ||
                length != origLength;
    }
}
```

聊到这里，你已经掌握了插件化开发的大部分只是细节了，要想更多地学习插件化在实际场景用的应用，请关注下一篇文章《插件化开发应用篇》

> 参考：
> https://www.cnblogs.com/ygj0930/p/6628429.html
> https://blog.csdn.net/sureyonder/article/details/5564181
> https://www.jianshu.com/p/0adea1b03d52
> https://stackoverflow.com/questions/39284030/java-classloader-difference-between-defining-loader-initiating-loader
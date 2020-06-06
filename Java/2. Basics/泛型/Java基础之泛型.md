
泛型在面向对象编程及各种设计模式中有非常广泛的应用。
泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。
这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。

那为什么要用泛型呢？

在 Java SE 1.5 之前，没有泛型的情况下，通过对类型 Object 的引用来实现参数的 "任意化"，缺点是要做显式的强制类型转换，而这种转换是要求开发对实际参数类型可以预知的情况下进行的。强制类型转换错误的

总原则：子类可以向父类转化，父类不能向子类转换（只能通过强转）

### 1. 什么是泛型

泛型是一种“代码模板”，可以用一套代码套用各种类型。

1. 编写一次，万能匹配

假设，ArrayList 中没有使用泛型：
```java
public class ArrayList{
    transient Object[] elementData;
    private int size;

    public boolean add(Object e) {
        // ...
    }

    public Object remove(int index) {
        // ...
    }

    public Object get(int index) {
        // ...
    }
}
```

如果用上述 ArrayList 存储 String 类型，会有如下缺点：
- 需要强制转型
- 不方便，易出错
```java
ArrayList list = new ArrayList();
list.add("Hello");
// 获取到 Object，必须强转型为 String
String first = (String) list.get(0);

list.add(new Integer(123));
// Error: ClassCastException
String second = (String) list.get(1);
```

要解决这个问题，我们可以为 String 单独编写一种 ArrayList：
```java
public class StringArrayList{
    transient String[] elementData;
    private int size;

    public boolean add(String e) {
        // ...
    }

    public String remove(int index) {
        // ...
    }

    public String get(int index) {
        // ...
    }
}
```
这样存入的必须是 String，取出的也一定是 String，不需要强制转型。

然而，如果要存储 Integer，还需要为 Integer 单独编写一种 ArrayList：
```java
public class IntegerArrayList{
    transient Integer[] elementData;
    private int size;

    public boolean add(Integer e) {
        // ...
    }

    public Integer remove(int index) {
        // ...
    }

    public Integer get(int index) {
        // ...
    }
}
```
Java 的类型有那么多，还要单独创建 LongArrayList、DoubleArrayList、PersonArrayList...

因此，我们必须把 ArrayList 编程一种模板，ArrayList<E>：
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    transient Object[] elementData;
    private int size;

    public boolean add(E e) {
        // ...
    }

    public E remove(int index) {
        // ...
    }

    public E get(int index) {
        // ...
    }
}
```

E 可以是任何 class，由编译器针对类型做检查：
```java
ArrayList<String> strList = new ArrayList<String>();
strList.add("hello"); // OK
String s = strList.get(0); // OK
strList.add(new Integer(123)); // compile error!
Integer n = strList.get(0); // compile error!
```

这样一来，就实现了编写一次，万能匹配，又通过编译器保证了类型安全，这就是泛型。


2. 向上转型

ArrayList<E> 实现了 List<E> 接口，它可以向上转型为 List<E>，E 不能变：
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    // ...
}

List<Integer> integerList = new ArrayList<Integer>();

// 向上转型，E 是不允许变的，编译就会报错。如果 list 加入了 Float 类型的数据，get 的时候将产生 ClassCastException
List<Number> numberList = integerList;
numberList.add(new Float(12.34));
Integer n = integerList.get(0);
```


3. 总结
泛型就是编写模板代码来适应任意类型；
泛型的好处是使用时不必对类型进行强制转换，它通过编译器对类型进行检查；
注意泛型的继承关系：可以把 ArrayList<Integer> 向上转型为List<Integer>（T不能变！），但不能把ArrayList<Integer>向上转型为ArrayList<Number>（T不能变成父类）。

### 2. 编写泛型 

#### 2.1 泛型类

Java标准库的 Map<K, V> 就是使用两种泛型类型的例子。它对 Key 使用一种类型，对 Value 使用另一种类型。
```java
@Data
public class Container<K,V>{
    private K key;
    private V value;

    public Container(K key,V value){
        this.key = key;
        this.value = value;
    }
}
```

#### 2.2 泛型接口

```java
public interface Generator<T>{
    public T generate();
}
```

```java
public class NumGenerator implements Generator<Integer> {
    int[] ages = {18,19,20};

    @Override
    public Integer generate(){
        Random random = new Random();
        return ages[random.nextInt(3)];
    }
}
```

#### 2.3 泛型方法

定义泛型方法时，必须在返回值前边加一个 <T>，来声明这是一个泛型 T，然后才可以用泛型作为方法的返回值。
为什么要使用泛型方法呢？因为泛型类要在实例化的时候就指明类型，如果想换一种类型，不得不重新 new 一次，可能不够灵活；而泛型方法可以在调用的时候指明类型，更加灵活。

```java
public class Generic{
    public <T> T getObject(Class<T> c) throws InstantiationException,IllegalAccessException{
        T t = c.newInstance();
        return t;
    }
 }
```


#### 2.4 extends 通配符

使用类似<? extends Number>通配符作为方法参数时表示：

方法内部可以调用获取Number引用的方法，例如：Number n = obj.getFirst();；

方法内部无法调用传入Number引用的方法（null除外），例如：obj.setFirst(Number n);。

即一句话总结：使用extends通配符表示可以读，不能写。

使用类似<T extends Number>定义泛型类时表示：

泛型类型限定为Number以及Number的子类。


#### 2.5 super 通配符



#### 2.6 extends 与 super 通配符对比

我们再回顾一下extends通配符。作为方法参数，<? extends T>类型和<? super T>类型的区别在于：
<? extends T>允许调用读方法T get()获取T的引用，但不允许调用写方法set(T)传入T的引用（传入null除外）；
<? super T>允许调用写方法set(T)传入T的引用，但不允许调用读方法T get()获取T的引用（获取Object除外）。
一个是允许读不允许写，另一个是允许写不允许读。

从 src 获取类型 T，写入到 dest 的类型 T：

```java
public class Collections {
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        int srcSize = src.size();
        if (srcSize > dest.size())
            throw new IndexOutOfBoundsException("Source does not fit in dest");

        if (srcSize < COPY_THRESHOLD ||
            (src instanceof RandomAccess && dest instanceof RandomAccess)) {
            for (int i=0; i<srcSize; i++)
                dest.set(i, src.get(i));
        } else {
            ListIterator<? super T> di=dest.listIterator();
            ListIterator<? extends T> si=src.listIterator();
            for (int i=0; i<srcSize; i++) {
                di.next();
                di.set(si.next());
            }
        }
    }
}
```
另一个好处是可以安全地把一个 List<Integer> 添加到 List<Number>，但是无法反过来添加。

- PECS原则

Producer Extends Consumer Super




#### 2.7 无限定通配符

Unbounded Wildcard Type，即只定义一个 ?
无限定通配符<?>很少使用，可以用<T>替换，同时它是所有<T>类型的超类。



### 3. 使用泛型

#### 3.1 常见的通配符

- ？  无限定通配符
- T   (type)，表示具体的一个 java 类型
- K,V (key,value)，表示键值对中的 key 和 value
- E   (element)，表示 Element

使用大写字母 A,B,C,D...X,Y,Z 定义的，就都是泛型，通常取类名的首字母，比较好对应。

#### 3.2 擦拭法

`<T>`不能是基本类型，例如 int，因为实际类型是 Object，Object 无法持有基本类型
无法取得带泛型的 Class
无法判断带泛型的 Class
不能实例化 T 类型

#### 3.3 泛型继承

子类可以获取父类的泛型类型`<T>`。

#### 3.4 不恰当的覆写方法

泛型方法要防止重复定义方法，例如：public boolean equals(T obj)；



### 4. 泛型和反射


Class<?> 与 Class<T> 的区别

Class<?> 是 Class<T> 的父类

















































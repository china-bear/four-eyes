
注解在一定程度上是在把元数据与源代码文件结合在一起，而不是保存在外部文档中这一大的趋势下所催生的。注解可以提供用来完整的描述程序所需的信息，而这些信息是无法用Java来表达的。
因此，注解存储有关程序的额外信息，是可以由编译器来测试和验证的。注解还可以用来生成描述符文件，甚至是新的类定义，并且有助于减轻编写“样板”代码的负担。通过使用注解，我们可以将这些元数据保存在Java源代码中，并利用 annotation API 为自己的注解构造处理工具，同时注解的优点还包括：更加干净易读的代码以及编译器类型检查等。

注解的使用场景：
- 提供信息给编译器：编译器可以利用注解来探测错误和警告信息
- 编译阶段时的处理：软件工具可以利用注解信息来生成代码，HTML文档或其他相应处理
- 运行时的处理：某些注解可以在程序运行时接受代码的提取

### 注解的分类

1. 按运行机制划分
源码注解：只在源码中存在，编译成 .class 文件就不存在了
编译时注解：在源码和 .class 文件中都存在，像前面的 @Override、@Deprecated、@SuppressWarnings 都属于编译时注解
运行时注解：在运行阶段还有作用，甚至会影响运行逻辑，像 @Autowired 就属于运行时注解，它会在程序运行时把你的成员变量自动的注入进来

2. 按来源划分
来自 JDK 的注解
来自第三方的注解
自定义注解

3. 元注解

### 元注解

负责注解的创建，是注解的注解。

1. @Target

表示注解可以用在什么地方。ElementType可以是：
- TYPE：类，接口，枚举类上
- FIELD：字段上，包括枚举实例
- METHOD：方法上
- PARAMETER：参数前
- CONSTRUCTOR：构造函数上
- LOCAL_VARIABLE：局部变量上
- ANNOTATION_TYPE：注解类上
- PACKAGE：包上
- TYPE_PARAMETER：
- TYPE_USE：
可以是某一个值或者以逗号分隔的形式指定多个值，如果想要将注解应用于所有的ElementType，也可以省去 @Target 元注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    /**
     * Returns an array of the kinds of elements an annotation type
     * can be applied to.
     * @return an array of the kinds of elements an annotation type
     * can be applied to
     */
    ElementType[] value();
}
```

2. @Retention

表示需要在什么级别上保留该注解信息。RetentionPolicy可以是：
- SOURCE：注解将被编译器丢弃
- CLASS：注解在class中可用，但会被VM丢弃
- RUNTIME：VM在运行期也将保留注解，因此可以通过反射机制读取注解信息

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    /**
     * Returns the retention policy.
     * @return the retention policy
     */
    RetentionPolicy value();
}
```

3. @Documented

将此注解中的元素包含到javadoc中。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Documented {
}
```

4. @Inherited

允许子类继承父类的注解。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Inherited {
}
```

5. @Repeatable

注解的值可以是多个，元素是一个容器注解。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Repeatable {
    /**
     * Indicates the <em>containing annotation type</em> for the
     * repeatable annotation type.
     * @return the containing annotation type
     */
    Class<? extends Annotation> value();
}
```


### 注解元素

1. 基本语法

使用 @interface 关键字定义注解，在注解上添加元注解。一般还要为注解添加元素，没有元素的注解称为标识注解。
注解只有成员变量，没有方法。注解的成员变量在注解的定义中以"无形参的方法"形式来声明，其方法名定义了该成员变量的名字，返回值定义了该成员变量的类型。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface TestAnnotation {

	int id() default -1;

	String msg() default "Hello";

	String value() default "";
}
```

2. 注解元素可用的类型

- 所有基本类型（int,float,boolean等）
- String
- Class
- enum
- Annotation
- 以上类型的数组

如果使用了其他类型，那编译器就会报错。也不允许使用任何包装类型。注解也可以作为元素的类型，也就是说注解可以嵌套。

3. 注解元素的默认值限制

编译器对注解元素的默认值有些过分挑剔。首先，注解元素不能有不确定的值。也就是说，注解元素要么具有默认值，要么在使用注解时设置元素值。


### 内置注解

所有的注解都继承自 java.lang.annotation.Annotation 接口。
```java
public interface Annotation {
    
    boolean equals(Object obj);

    int hashCode();

    String toString();

    Class<? extends Annotation> annotationType();
}
```

1. @Override

表示当前的方法定义将覆盖超类中的方法。如果不小心拼写错误或者方法签名对不上被覆盖的方法，编译器就会发出错误提示。
```java
package java.lang;
import java.lang.annotation.*;
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

执行如下命令：
```shell
$ javac Override.java
$ javap -c Override.class
```

得到如下内容：
```txt
Compiled from "Override.java"
public interface java.lang.Override extends java.lang.annotation.Annotation {
}
```

由此可以看出，注解的本质就是一个继承了 Annotation 接口的接口，是一种典型的标记式注解。
一旦编译器检测到某个方法被修饰了 @Override 注解，编译器就会检查当前方法的方法签名是否真正重写了父类的某个方法，也就是比较父类中是否具有一个同样的方法签名，如果没有，自然不能编译通过。
编译器只能识别已经熟知的注解类，比如 JDK 内置的几个注解，而我们自定义的注解，编译器是不会知道这个注解的作用的，当然也不知道应该如何处理。


2. @Deprecated

依然是一种标记式注解，永久存在，可以修饰所有类型，被标记的类、方法、字段等已经不再被推荐使用了，可能下一个版本就会删除。当然，编译器并不会强制要求你做什么，只是会在对象上画出一道线，建议你使用某个替代者。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {
}
```


3. @SuppressWarnings

抑制告警。它有一个 value 属性需要主动传值，传入需要被抑制的警告类型。
```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```

如下 Date 的构造函数是过时的，在 main() 方法上加上 @SuppressWarning(value = "deprecated") 注解后，编译器就不会再对这种告警进行检查了。
```java
@SuppressWarning(value = "deprecated")
public static void main(String[] args) {
    Date date = new Date(2018, 7, 11);
}
```

### 注解的提取

解析一个注解通常有两种形式：
- 一种是编译期直接扫描：编译器在对java代码编译字节码的过程中会检测到某个类或方法被一些注解修饰，它就会对这些注解进行某些处理。
- 一种是运行期反射。


上文中有创建注解 `TestAnnotation` ，下面我们来写一个注解的提取类 Test：
```java
@TestAnnotation("defaultValue")
public class Test {

	public static void main(String[] args) {
		// 注解通过反射获取，通过 Class 对象的 isAnnotationPresent() 方法判断它是否应用了某个注解
		boolean hasAnnotation = Test.class.isAnnotationPresent(TestAnnotation.class);
		if (hasAnnotation) {
			// 通过 getAnnotation() 方法来获取 Annotation 对象实例
			TestAnnotation testAnnotation = Test.class.getAnnotation(TestAnnotation.class);
			System.out.println("id:" + testAnnotation.id());
			System.out.println("msg:" + testAnnotation.msg());
		}

	}
}
```

我们前面说过，注解本质上是继承了 Annotation 接口的接口，而当你通过反射，也就是使用 getAnnotation 方法去获取一个注解类实例的时候，JDK 是通过动态代理生成了一个实现自定义注解（接口）的代理类。

下面我们来做一个实验：

首先，运行 Test 类之前，先设置如下 VM 参数，让其生成代理类 class 文件：
```shell
/* jdk动态代理 设置此系统属性,让JVM生成的Proxy类写入文件.保存路径为：com/sun/proxy(如果不存在请手工创建) */
-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true
/* cglib动态代理 设置此系统属性,让JVM生成的Proxy类写入文件.保存路径为：com/sun/proxy(如果不存在请手工创建) */
-Dcglib.debugLocation=com/sun/proxy
```

然后，运行 Test 类并反编译代理类的 class 文件：
```shell
$ cd ./com/sun/proxy
$ javap -c \$Proxy1.class > Proxy1
```

最后，查看 Proxy1 文件内容，可见代理类实现接口 TestAnnotation 并重写其所有方法，包括id()、msg()、value()以及接口 TestAnnotation 从 Annotation 接口继承而来的方法。
查看 value() 方法的调用步骤，是通过代理类的 InvocationHandler.invoke 方法调用返回注解元素 value 的值。
```txt
public final class com.sun.proxy.$Proxy1 extends java.lang.reflect.Proxy implements org.apache.flink.annotation.TestAnnotation {
  public com.sun.proxy.$Proxy1(java.lang.reflect.InvocationHandler) throws ;
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #8                  // Method java/lang/reflect/Proxy."<init>":(Ljava/lang/reflect/InvocationHandler;)V
       5: return

  public final boolean equals(java.lang.Object) throws ;
    ......

  public final java.lang.String toString() throws ;
    ......

  public final java.lang.String msg() throws ;
    ......

  public final java.lang.Class annotationType() throws ;
    ......

  public final int id() throws ;
    ......

  public final int hashCode() throws ;
    ......

  public final java.lang.String value() throws ;
    Code:
       0: aload_0
       1: getfield      #16                 // Field java/lang/reflect/Proxy.h:Ljava/lang/reflect/InvocationHandler;
       4: aload_0
       5: getstatic     #81                 // Field m3:Ljava/lang/reflect/Method;
       8: aconst_null
       9: invokeinterface #28,  4           // InterfaceMethod java/lang/reflect/InvocationHandler.invoke:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;
      14: checkcast     #52                 // class java/lang/String
      17: areturn
      18: athrow
      19: astore_1
      20: new           #42                 // class java/lang/reflect/UndeclaredThrowableException
      23: dup
      24: aload_1
      25: invokespecial #45                 // Method java/lang/reflect/UndeclaredThrowableException."<init>":(Ljava/lang/Throwable;)V
      28: athrow
    Exception table:
       from    to  target type
           0    18    18   Class java/lang/Error
           0    18    18   Class java/lang/RuntimeException
           0    18    19   Class java/lang/Throwable

  static {} throws ;
    ......
}
```

其中的关键是，$Proxy1构造函数的入参 InvocationHandler 是什么？
这里的 InvocationHandler 指的就是 AnnotationInvocationHandler，它是 Java 中专门用于处理注解的 handler，下面就来让我们看看这个类的实现。
memberValues 存放的是注解元素的键值对，invoke() 执行注解类中的各个元素值方法。
```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private static final long serialVersionUID = 6182022883658399397L;
    private final Class<? extends Annotation> type;
    /**
     * 注解元素属性的键值对
     */
    private final Map<String, Object> memberValues;

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        this.type = type;
        this.memberValues = memberValues;
    }

    /**
     * 代理类代理了 TestAnnotation 接口中的所有方法
     */
    public Object invoke(Object proxy, Method method, Object[] args) {
        String member = method.getName();
        Class<?>[] paramTypes = method.getParameterTypes();

        // Handle Object and Annotation methods
        // 如果当前调用的是 toString、equals、hashCode、annotationType。AnnotationInvocationHandler 实例中已经预定义好了这些方法的实现，直接调用即可。
        if (member.equals("equals") && paramTypes.length == 1 &&
            paramTypes[0] == Object.class)
            return equalsImpl(args[0]);
        assert paramTypes.length == 0;
        if (member.equals("toString"))
            return toStringImpl();
        if (member.equals("hashCode"))
            return hashCodeImpl();
        if (member.equals("annotationType"))
            return type;

        // Handle annotation member accessors
        // 从我们注解的 map 中获取这个注解属性对应的值，即通过方法名返回注解属性值。
        Object result = memberValues.get(member);

        if (result == null)
            throw new IncompleteAnnotationException(type, member);

        if (result instanceof ExceptionProxy)
            throw ((ExceptionProxy) result).generateException();

        if (result.getClass().isArray() && Array.getLength(result) != 0)
            result = cloneArray(result);

        return result;
    }

}
```

### 自定义注解


1. 可重复注解

创建容器注解 Persons，容器注解本身也是一个注解，是用来存放其他注解的地方。它必须要有一个 value 属性，属性类型是一个被 @Repeatable 注解过的注解数组。
```java
public @interface Persons {
	Person[] value();
}
```

使用 @Repeatable 注解了 Person，而 @Repeatable 后面括号中的类是一个容器注解。
```java
@Repeatable(Persons.class)
public @interface Person {
	String role();
}
```

给 Superman 这个类贴上多个角色标签。
```java
@Person(role="Painter")
@Person(role="Musician")
@Person(role="Actor")
public class Superman {
}
```

2. 测试用例注解
实现一个注解，用来跟踪一个项目中的用例。如果一个方法实现了某个用例的需求，那么可以为此方法加上该注解。于是，项目经理通过计算已经实现的用例，就可以很好的掌控项目的进展。而且把实现方法和用例绑定，如果要更新或修改系统的业务逻辑，维护该项目的开发人员也可以很容易的在代码中找到对应的用例。

定义 UseCase 注解，id 表示用例编号，description 设置了默认值。
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseCase {
    /**
	 * 用例id
 	 */
	int id();

	/**
	 * 用例描述
	 */
	String description() default "no description";
}
```

定义需求实现类 PasswordUtils，每一个方法都对应一个需求用例。
```java
public class PasswordUtils {

    @UseCase(id = 47 , description = "Passwords must contain at least one numeric")
    public boolean validatePassword(String password){
        return password.matches("\\w*\\d\\w*");
    }


    @UseCase(id = 48)
    public String encryptPassword(String password){
        return new StringBuilder(password).reverse().toString();
    }

    @UseCase(id = 49,description = "New passwords can't equal previously used ones")
    public boolean checkForNewPassword(List<String> prevPasswords, String password){
        return !prevPasswords.contains(password);
    }

}
```

如果没有用来读取注解的工具，那注解也不会比注释更有用。使用注解的过程中，很重要的一部分就是创建与使用注解处理器。
下面实现了一个非常简单的注解处理器 UseCaseTracker ，将用它来读取 PasswordUtils 类，并使用反射机制查找 @UseCase 注解。
我们提供了一组 id 值，然后它会列出在 PasswordUtils 中找到的用例，以及缺失的用例。
```java
public class UseCaseTracker {

	public static void trackUseCases(List<Integer> useCases, Class<?> cl) {
		
		for (Method m : cl.getDeclaredMethods()) {
			// 返回指定类型的注解对象
			UseCase uc = m.getAnnotation(UseCase.class);
			if (uc != null) {
				System.out.println("Found Use Case: " + uc.id() + " " + uc.description());
				useCases.remove(new Integer(uc.id()));
			}
		}
		for (Integer i : useCases) {
			System.out.println("Warning: Missing use case-" + i);
		}
	}

	public static void main(String[] args) {
		List<Integer> useCases = new ArrayList<>();
		Collections.addAll(useCases, 47, 48, 49, 50);
		trackUseCases(useCases, PasswordUtils.class);
	}
}
```
运行结果：
```
Found Use Case: 47 Passwords must contain at least one numeric
Found Use Case: 48 no description
Found Use Case: 49 New passwords can't equal previously used ones
Warning: Missing use case-50
```

3. 利用注解生成SQL语句

定义表名注解，它告诉处理器，你需要把我这个类生成一个数据库 DDL 语句。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DBTable {
	/**
	 * 数据库表表名
	 */
	String name() default "";
}
```

定义数据库表字段约束的注解：是否为主键，是否可以为空，唯一性约束。
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Constraints {
    boolean primaryKey() default false;
    boolean allowNull() default true;
    boolean unique() default false;
}
```

定义表字段类型为 String 的注解：字符串长度，字段名， 字段约束。这里的字段约束就用到了嵌套注解的语法。
```java
@Target(value = ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLString {

    int len() default 0;
    String name() default "";
    Constraints constraints() default @Constraints;
}
```

定义表字段类型为 Integer 的注解：字段名，字段约束。
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLInteger {
    String name() default "";
    Constraints constraints() default @Constraints;
}
```

定义一个 Member 类，应用了以上定义的注解。类的注解 @DBTable 给定了值 MEMBER，它将会用来作为表的名字。字段属性 firstName 和 lastName 都被注解为 @SQLString 类型，并分别设置了长度为 30 和 50。
```java
@DBTable(name = "MEMBER")
public class Member {

	@SQLString(len = 30)
	String firstName;

	@SQLString(len = 50)
	String lastName;

	@SQLInteger
	Integer age;

	@SQLString(len = 30, constraints = @Constraints(primaryKey = true))
	String handle;

	static int memberCount;

	public String getFirstName() {
		return firstName;
	}

	public String getLastName() {
		return lastName;
	}

	public Integer getAge() {
		return age;
	}

	public String getHandle() {
		return handle;
	}

	@Override
	public String toString() {
		return handle;
	}
}
```

实现处理器 TableCreator ，它将读取一个类文件，检查其上的数据库表注解，并生成用来创建数据库表的 SQL 语句。
```java
public class TableCreator {

	public static void main(String[] args) throws Exception {

		String className = Member.class.getName();
		Class<?> cl = Class.forName(className);

		// 检查类上是否带有 @DBTable 注解
		DBTable dbtable = cl.getAnnotation(DBTable.class);
		if (dbtable == null) {
			System.out.println("No DbTable annotations in class " + className);
		}

        // 提取 @DBTable 注解的 name
		String tableName = dbtable.name();
		// If the name is empty , use the Class name:
		if (tableName.length() < 1) {
			tableName = cl.getName().toUpperCase();
		}

		List<String> columnDefs = new ArrayList<>();
		// 遍历 Member 类的所有字段
		for (Field field : cl.getDeclaredFields()) {
			String columnName;

			// 获取字段属性上的所有注解
			Annotation[] annotations = field.getDeclaredAnnotations();
			if (annotations.length < 1) {
				continue; // Not a db table column
			}

			if (annotations[0] instanceof SQLInteger) {
				// 处理 @SQLInteger 注解的属性字段
				SQLInteger sInt = (SQLInteger) annotations[0];
				// Use field name if name not specified
				if (sInt.name().length() < 1) {
					columnName = field.getName().toUpperCase();
				} else {
					columnName = sInt.name();
				}
				columnDefs.add(columnName + " INT" + getConstraints(sInt.constraints()));
			} else if (annotations[0] instanceof SQLString) {
				// 处理 @SQLString 注解的属性字段
				SQLString sString = (SQLString) annotations[0];
				// Use field name if name not specified.
				if (sString.name().length() < 1) {
					columnName = field.getName().toUpperCase();
				} else {
					columnName = sString.name();
				}
				columnDefs.add(columnName + " VARCHAR(" + sString.len() + ")" + getConstraints(sString.constraints()));
			}
		}

		StringBuilder createCommand = new StringBuilder("CREATE TABLE " + tableName + "(");
		for (String columnDef : columnDefs) {
			createCommand.append("\n    " + columnDef + ",");
		}
		// Remove trailing comma
		String tableCreate = createCommand.substring(0, (createCommand.length() - 1)) + ");";
		System.out.println("Table.Creation SQL for " + className + " is :\n " + tableCreate);
	}

	private static String getConstraints(Constraints con) {
		String constraints = "";
		if (!con.allowNull()) {
			constraints += " NOT NULL";
		}
		if (con.primaryKey()) {
			constraints += " PRIMARY KEY";
		}
		if (con.unique()) {
			constraints += " UNIQUE";
		}
		return constraints;
	}
}
```


运行结果：
```
Table.Creation SQL for org.apache.flink.annotation.dbtable.Member is :
 CREATE TABLE MEMBER(
    FIRSTNAME VARCHAR(30),
    LASTNAME VARCHAR(50),
    AGE INT,
    HANDLE VARCHAR(30) PRIMARY KEY);
```














































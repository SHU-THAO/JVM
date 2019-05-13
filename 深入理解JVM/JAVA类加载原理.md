# Java类加载原理解析

​	每个开发人员对java.lang.ClassNotFoundExcetpion这个异常肯定都不陌生，这个异常背后涉及到的是Java技术体系中的类加载机制。本文简述了JVM三种预定义类加载器，即启动类加载器、扩展类加载器和系统类加载器，并介绍和分析它们之间的关系和类加载所采用的双亲委派机制，给出并分析了与Java类加载原理相关的若干问题。



## 引子

​	每个开发人员对java.lang.ClassNotFoundExcetpion这个异常肯定都不陌生，其实，这个异常背后涉及到的是Java技术体系中的类加载。`Java类加载机制`虽然和大部分开发人员直接打交道的机会不多，但是对其机理的理解有助于排查程序出现的类加载失败等技术问题，对理解Java虚拟机的连接模型和Java语言的动态性都有很大帮助。

​	

## Java 虚拟机类加载器结构简述

### 1、JVM三种预定义类型类加载器

​	当JVM启动的时候，Java开始使用如下三种类型的类加载器：

- `启动（Bootstrap）类加载器`：启动类加载器是用本地代码实现的类加载器，它负责将JAVA_HOME/lib下面的核心类库或-Xbootclasspath选项指定的jar包等虚拟机识别的类库加载到内存中。由于启动类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用。具体可由启动类加载器加载到的路径可通过`System.getProperty(“sun.boot.class.path”)`查看。
- `扩展（Extension）类加载器`：扩展类加载器是由Sun的ExtClassLoader （sun.misc.Launcher$ExtClass Loader） 实现的，它负责将JAVA_HOME /lib/ext或者由系统变量-Djava.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器，具体可由扩展类加载器加载到的路径可通过`System.getProperty("java.ext.dirs")`查看。
- `系统（System）类加载器`：系统类加载器是由 Sun 的 AppClassLoader（sun.misc.Launcher$AppClass Loader）实现的，它负责将用户类路径(java -classpath或-Djava.class.path变量所指的目录，即当前类所在路径及其引用的第三方类库的路径，如第四节中的问题6所述)下的类库加载到内存中。开发者可以直接使用系统类加载器，具体可由系统类加载器加载到的路径可通过`System.getProperty("java.class.path")`查看。

Ps: 除了以上列举的三种类加载器，还有一种比较特殊的类型就是线程上下文类加载器，另外单独学习。

### 2、类加载双亲委派机制介绍和分析

​	JVM在加载类时默认采用的是双亲委派机制。通俗的讲，就是某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，**依次递归** (本质上就是loadClass函数的递归调用)，因此所有的加载请求最终都应该传送到顶层的启动类加载器中。如果父类加载器可以完成这个类加载请求，就成功返回；只有当父类加载器无法完成此加载请求时，子加载器才会尝试自己去加载。

​	事实上，大多数情况下，越基础的类由越上层的加载器进行加载，因为这些基础类之所以称为“基础”，是因为它们总是作为被用户代码调用的API（当然，也存在基础类回调用户用户代码的情形，即破坏双亲委派模型的情形）。关于虚拟机默认的双亲委派机制，我们可以从系统类加载器和扩展类加载器为例作简单分析。

![æ åæ©å±ç±"å è½½å¨ç"§æ¿å±æ¬¡å¾-17.2kB](C:\Users\12084\Desktop\深入理解JVM\扩展Extension类加载器.png)

![ç³"ç"ç±"å è½½å¨ç"§æ¿å±æ¬¡å¾-16.4kB](C:\Users\12084\Desktop\深入理解JVM\系统(System)类加载器.png)

​	上面两张图分别是扩展类加载器继承层次图和系统类加载器继承层次图。通过这两张图我们可以看出，扩展类加载器和系统类加载器均是继承自 java.lang.ClassLoader抽象类。下面我们就看简要介绍一下抽象类 java.lang. ClassLoader 中几个最重要的方法：

``` java
//加载指定名称（包括包名）的二进制类型，供用户调用的接口  
public Class<?> loadClass(String name) throws ClassNotFoundException{ … }  
  
//加载指定名称（包括包名）的二进制类型，同时指定是否解析（但是这里的resolve参数不一定真正能达到解析的效果），供继承用  
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException{ … }  
  
//findClass方法一般被loadClass方法调用去加载指定名称类，供继承用  
protected Class<?> findClass(String name) throws ClassNotFoundException { … }  
  
//定义类型，一般在findClass方法中读取到对应字节码后调用，final的，不能被继承  
//这也从侧面说明：JVM已经实现了对应的具体功能，解析对应的字节码，产生对应的内部数据结构放置到方法区，所以无需覆写，直接调用就可以了）  
protected final Class<?> defineClass(String name, byte[] b, int off, int len) throws ClassFormatError{ … }  
```

​	通过进一步分析标准扩展类加载器和系统类加载器的代码以及其公共父类（java.net.URLClassLoader和java.security.SecureClassLoader）的代码可以看出，都没有覆写java.lang.ClassLoader中默认的加载委派规则 — loadClass（…）方法。既然这样，我们就可以从java.lang.ClassLoader中的loadClass（String name）方法的代码中分析出虚拟机默认采用的双亲委派机制到底是什么模样：

``` java
public Class<?> loadClass(String name) throws ClassNotFoundException {  
    return loadClass(name, false);  
}  
  
protected synchronized Class<?> loadClass(String name, boolean resolve)  
        throws ClassNotFoundException {  
  
    // 首先判断该类型是否已经被加载  
    Class c = findLoadedClass(name);  
    if (c == null) {  
        //如果没有被加载，就委托给父类加载或者委派给启动类加载器加载  
        try {  
            if (parent != null) {  
                //如果存在父类加载器，就委派给父类加载器加载  
                c = parent.loadClass(name, false);  
            } else {    // 递归终止条件
                // 由于启动类加载器无法被Java程序直接引用，因此默认用 null 替代
                // parent == null就意味着由启动类加载器尝试加载该类，  
                // 即通过调用 native方法 findBootstrapClass0(String name)加载  
                c = findBootstrapClass0(name);  
            }  
        } catch (ClassNotFoundException e) {  
            // 如果父类加载器不能完成加载请求时，再调用自身的findClass方法进行类加载，若加载成功，findClass方法返回的是defineClass方法的返回值
            // 注意，若自身也加载不了，会产生ClassNotFoundException异常并向上抛出
            c = findClass(name);  
        }  
    }  
    if (resolve) {  
        resolveClass(c);  
    }  
    return c;  
}  
```

​	通过上面的代码分析，我们可以对JVM采用的双亲委派类加载机制有了更直接的认识。下面我们就接着分析一下启动类加载器、标准扩展类加载器和系统类加载器三者之间的关系。可能大家已经从各种资料上面看到了如下类似的一幅图片：

![ç±"å è½½å¨é"è®¤å§æ´¾å³ç³"å¾-11.2kB](C:\Users\12084\Desktop\深入理解JVM\双亲委派模型.png)

​	上面图片给人的直观印象是系统类加载器的父类加载器是标准扩展类加载器，标准扩展类加载器的父类加载器是启动类加载器，下面我们就用代码具体测试一下：

``` java
public class LoaderTest {  
  
    public static void main(String[] args) {  
        try {  
            System.out.println(ClassLoader.getSystemClassLoader());  
            System.out.println(ClassLoader.getSystemClassLoader().getParent());  
            System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}/* Output: 
        sun.misc.Launcher$AppClassLoader@6d06d69c  
        sun.misc.Launcher$ExtClassLoader@70dea4e  
        null  
 *///:~
```

​	通过以上的代码输出，我们知道：通过java.lang.ClassLoader.getSystemClassLoader()可以直接获取到系统类加载器 ，并且可以判定系统类加载器的父加载器是标准扩展类加载器，但是我们试图获取标准扩展类加载器的父类加载器时却得到了null。事实上，由于启动类加载器无法被Java程序直接引用，因此JVM默认直接使用 null 代表启动类加载器。我们还是借助于代码分析一下，首先看一下java.lang.ClassLoader抽象类中默认实现的两个构造函数：

``` java
protected ClassLoader() {  
    SecurityManager security = System.getSecurityManager();  
    if (security != null) {  
        security.checkCreateClassLoader();  
    }  
    //默认将父类加载器设置为系统类加载器，getSystemClassLoader()获取系统类加载器  
    this.parent = getSystemClassLoader();  
    initialized = true;  
}  

protected ClassLoader(ClassLoader parent) {  
    SecurityManager security = System.getSecurityManager();  
    if (security != null) {  
        security.checkCreateClassLoader();  
    }  
    //强制设置父类加载器  
    this.parent = parent;  
    initialized = true;  
}  
```

紧接着，我们再看一下ClassLoader抽象类中parent成员的声明：

``` java
// The parent class loader for delegation  
private ClassLoader parent; 
```

声明为私有变量的同时并没有对外提供可供派生类访问的public或者protected设置器接口（对应的setter方法），结合前面的测试代码的输出，我们可以推断出：

1．系统类加载器（AppClassLoader）调用ClassLoader(ClassLoader parent)构造函数将父类加载器设置为标准扩展类加载器(ExtClassLoader)。（因为如果不强制设置，默认会通过调用getSystemClassLoader()方法获取并设置成系统类加载器，这显然和测试输出结果不符。）

2．扩展类加载器（ExtClassLoader）调用ClassLoader(ClassLoader parent)构造函数将父类加载器设置为null（null 本身就代表着引导类加载器）。（因为如果不强制设置，默认会通过调用getSystemClassLoader()方法获取并设置成系统类加载器，这显然和测试输出结果不符。）

事实上，这就是启动类加载器、标准扩展类加载器和系统类加载器之间的委派关系。

3、类加载双亲委派示例

以上已经简要介绍了虚拟机默认使用的启动类加载器、标准扩展类加载器和系统类加载器，并以三者为例结合JDK代码对JVM默认使用的双亲委派类加载机制做了分析。下面我们就来看一个综合的例子，首先在IDE中建立一个简单的java应用工程，然后写一个简单的JavaBean如下：

``` java
package classloader.test.bean;  

public class TestBean {  
      
    public TestBean() { }  
}  
```

在现有当前工程中另外建立一个测试类（ClassLoaderTest.java）内容如下：

``` java
package classloader.test.bean;  
  
public class ClassLoaderTest {  
  
    public static void main(String[] args) {  
        try {  
            //查看当前系统类路径中包含的路径条目  
            System.out.println(System.getProperty("java.class.path"));  
            //调用加载当前类的类加载器（这里即为系统类加载器）加载TestBean  
            Class typeLoaded = Class.forName("classloader.test.bean.TestBean");  
            //查看被加载的TestBean类型是被那个类加载器加载的  
            System.out.println(typeLoaded.getClassLoader());  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}/* Output: 
        I:\AlgorithmPractice\TestClassLoader\bin
        sun.misc.Launcher$AppClassLoader@6150818a
 *///:~  
```

将当前工程输出目录下的TestBean.class打包进test.jar剪贴到<Java_Runtime_Home>/lib/ext目录下（现在工程输出目录下和JRE扩展目录下都有待加载类型的class文件）。再运行测试一测试代码，结果如下：

``` java
    I:\AlgorithmPractice\TestClassLoader\bin
    sun.misc.Launcher$ExtClassLoader@15db9742
```

对比上面的两个结果，我们明显可以验证前面说的双亲委派机制：系统类加载器在接到加载classloader.test.bean .TestBean类型的请求时，首先将请求委派给父类加载器（标准扩展类加载器），标准扩展类加载器抢先完成了加载请求。

最后，将test.jar拷贝一份到<Java_Runtime_Home>/lib下，运行测试代码，输出如下：

``` java
    I:\AlgorithmPractice\TestClassLoader\bin
    sun.misc.Launcher$ExtClassLoader@15db9742
```

可以看到，后两次输出结果一致。那就是说，放置到<Java_Runtime_Home>/lib目录下的TestBean对应的class字节码并没有被加载，这其实和前面讲的双亲委派机制并不矛盾。虚拟机出于安全等因素考虑，不会加载<JAVA_HOME>/lib目录下存在的陌生类。换句话说，虚拟机只加载<JAVA_HOME>/lib目录下它可以识别的类。因此，开发者通过将要加载的非JDK自身的类放置到此目录下期待启动类加载器加载是不可能的。做个进一步验证，删除<JAVA_HOME>/lib/ext目录下和工程输出目录下的TestBean对应的class文件，然后再运行测试代码，则将会有ClassNotFoundException异常抛出。有关这个问题，大家可以在java.lang.ClassLoader中的loadClass(String name, boolean resolve)方法中设置相应断点进行调试，会发现findBootstrapClass0()会抛出异常，然后在下面的findClass方法中被加载，当前运行的类加载器正是扩展类加载器（sun.misc.Launcher$ExtClassLoader），这一点可以通过JDT中变量视图查看验证。



## Java 程序动态扩展方式

​	Java的连接模型允许用户运行时扩展引用程序，既可以通过当前虚拟机中预定义的加载器加载编译时已知的类或者接口，又允许用户自行定义类装载器，在运行时动态扩展用户的程序。通过用户自定义的类装载器，你的程序可以加载在编译时并不知道或者尚未存在的类或者接口，并动态连接它们并进行有选择的解析。运行时动态扩展java应用程序有如下两个途径：

### 1、反射 (调用java.lang.Class.forName(…)加载类)

​	这个方法其实在前面已经讨论过，在后面的问题2解答中说明了该方法调用会触发哪个类加载器开始加载任务。这里需要说明的是多参数版本的forName(…)方法：

``` java
public static Class<?> forName(String name, boolean initialize, ClassLoader loader) throws ClassNotFoundException  
```

​	这里的initialize参数是很重要的，它表示在加载同时是否完成初始化的工作（说明：单参数版本的forName方法默认是完成初始化的）。有些场景下需要将initialize设置为true来强制加载同时完成初始化，例如典型的就是加载数据库驱动问题。因为JDBC驱动程序只有被注册后才能被应用程序使用，这就要求驱动程序类必须被初始化，而不单单被加载。

``` java
// 加载并实例化JDBC驱动类
Class.forName(driver);
 
 // JDBC驱动类的实现
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }
	// 将initialize设置为true来强制加载同时完成初始化，实现驱动注册
    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can\'t register driver!");
        }
    }
}
```



### 2、用户自定义类加载器

​	通过前面的分析，我们可以看出，除了和本地实现密切相关的启动类加载器之外，包括标准扩展类加载器和系统类加载器在内的所有其他类加载器我们都可以当做自定义类加载器来对待，唯一区别是是否被虚拟机默认使用。前面的内容中已经对java.lang.ClassLoader抽象类中的几个重要的方法做了介绍，这里就简要叙述一下一般用户自定义类加载器的工作流程（可以结合后面问题解答一起看）：

1. 首先检查请求的类型是否已经被这个类装载器装载到命名空间中了，如果已经装载，直接返回；否则转入步骤2；
2. 委派类加载请求给父类加载器（更准确的说应该是双亲类加载器，真实虚拟机中各种类加载器最终会呈现树状结构），如果父类加载器能够完成，则返回父类加载器加载的Class实例；否则转入步骤3；
3. 调用本类加载器的findClass（…）方法，试图获取对应的字节码。如果获取的到，则调用defineClass（…）导入类型到方法区；如果获取不到对应的字节码或者其他原因失败， 向上抛异常给loadClass（…）， loadClass（…）转而调用findClass（…）方法处理异常，直至完成递归调用。

必须指出的是，这里所说的自定义类加载器是指JDK1.2以后版本的写法，即不覆写改变java.lang.loadClass(…)已有委派逻辑情况下。整个加载类的过程如下图：

![èªå®ä¹ç±"å è½½å¨å è½½ç±"çè¿ç¨-54.2kB](C:\Users\12084\Desktop\深入理解JVM\双亲委派模型2.png)



## 常见问题分析

**1、由不同的类加载器加载的指定类还是相同的类型吗？**

​	在Java中，一个类用其完全匹配类名(fully qualified class name)作为标识，这里指的完全匹配类名包括包名和类名。但在JVM中，一个类用其全名 和 一个ClassLoader的实例作为唯一标识，不同类加载器加载的类将被置于不同的命名空间。我们可以用两个自定义类加载器去加载某自定义类型（注意不要将自定义类型的字节码放置到系统路径或者扩展路径中，否则会被系统类加载器或扩展类加载器抢先加载），然后用获取到的两个Class实例进行java.lang.Object.equals（…）判断，将会得到不相等的结果，如下所示：

``` java
public class TestBean {

	public static void main(String[] args) throws Exception {
	    // 一个简单的类加载器，逆向双亲委派机制
	    // 可以加载与自己在同一路径下的Class文件
		ClassLoader myClassLoader = new ClassLoader() {
			@Override
			public Class<?> loadClass(String name)
					throws ClassNotFoundException {
				try {
					String filename = name.substring(name.lastIndexOf(".") + 1)
							+ ".class";
					InputStream is = getClass().getResourceAsStream(filename);
					if (is == null) {
						return super.loadClass(name);   // 递归调用父类加载器
					}
					byte[] b = new byte[is.available()];
					is.read(b);
					return defineClass(name, b, 0, b.length);
				} catch (Exception e) {
					throw new ClassNotFoundException(name);
				}
			}
		};

		Object obj = myClassLoader.loadClass("classloader.test.bean.TestBean")
				.newInstance();
		System.out.println(obj.getClass());
		System.out.println(obj instanceof classloader.test.bean.TestBean);
	}
}/* Output: 
        class classloader.test.bean.TestBean
        false  
 *///:~    
```

​	我们发现，obj 确实是类classloader.test.bean.TestBean实例化出来的对象，但当这个对象与类classloader.test.bean.TestBean做所属类型检查时却返回了false。这是因为虚拟机中存在了两个TestBean类，一个是由系统类加载器加载的，另一个则是由我们自定义的类加载器加载的，虽然它们来自同一个Class文件，但依然是两个独立的类，因此做所属类型检查时返回false。



**2、在代码中直接调用Class.forName(String name)方法，到底会触发那个类加载器进行类加载行为？**

Class.forName(String name)默认会使用调用类的类加载器来进行类加载。我们直接来分析一下对应的jdk的代码：

``` java
//java.lang.Class.java  
publicstatic Class<?> forName(String className) throws ClassNotFoundException {  
    return forName0(className, true, ClassLoader.getCallerClassLoader());  
}  
  
//java.lang.ClassLoader.java  
// Returns the invoker's class loader, or null if none.  
static ClassLoader getCallerClassLoader() {  
    // 获取调用类（caller）的类型  
    Class caller = Reflection.getCallerClass(3);  
    // This can be null if the VM is requesting it  
    if (caller == null) {  
        return null;  
    }  
    // 调用java.lang.Class中本地方法获取加载该调用类（caller）的ClassLoader  
    return caller.getClassLoader0();  
}  
  
//java.lang.Class.java  
//虚拟机本地实现，获取当前类的类加载器，前面介绍的Class的getClassLoader()也使用此方法  
native ClassLoader getClassLoader0(); 
```



**3、在编写自定义类加载器时，如果没有设定父加载器，那么父加载器是谁？**

​	前面讲过，在不指定父类加载器的情况下，默认采用系统类加载器。可能有人觉得不明白，现在我们来看一下JDK对应的代码实现。众所周知，我们编写自定义的类加载器直接或者间接继承自java.lang.ClassLoader抽象类，对应的无参默认构造函数实现如下：

``` java
//摘自java.lang.ClassLoader.java  
protected ClassLoader() {  
    SecurityManager security = System.getSecurityManager();  
    if (security != null) {  
        security.checkCreateClassLoader();  
    }  
    this.parent = getSystemClassLoader();  
    initialized = true;  
} 
```

我们再来看一下对应的getSystemClassLoader()方法的实现：

``` java
private static synchronized void initSystemClassLoader() {  
    //...  
    sun.misc.Launcher l = sun.misc.Launcher.getLauncher();  
    scl = l.getClassLoader();  
    //...  
}  
```

我们可以写简单的测试代码来测试一下：

``` java
System.out.println(sun.misc.Launcher.getLauncher().getClassLoader());  
```

本机对应输出如下：

```java
sun.misc.Launcher$AppClassLoader@73d16e93 
```

所以，我们现在可以相信当自定义类加载器没有指定父类加载器的情况下，默认的父类加载器即为系统类加载器。

同时，我们可以得出如下结论：即使用户自定义类加载器不指定父类加载器，那么，同样可以加载如下三个地方的类：

- <Java_Runtime_Home>/lib下的类；
- <Java_Runtime_Home>/lib/ext下或者由系统变量java.ext.dir指定位置中的类；
- 当前工程类路径下或者由系统变量java.class.path指定位置中的类。



**4、在编写自定义类加载器时，如果将父类加载器强制设置为null，那么会有什么影响？如果自定义的类加载器不能加载指定类，就肯定会加载失败吗？**

JVM规范中规定如果用户自定义的类加载器将父类加载器强制设置为null，那么会自动将启动类加载器设置为当前用户自定义类加载器的父类加载器（这个问题前面已经分析过了）。

Ps：问题3和问题4的推断结论是基于用户自定义的类加载器本身延续了java.lang.ClassLoader.loadClass（…）默认委派逻辑，如果用户对这一默认委派逻辑进行了改变，以上推断结论就不一定成立了，详见问题 5。



**5、编写自定义类加载器时，一般有哪些注意点？**

1)、一般尽量不要覆写已有的loadClass(…)方法中的委派逻辑（Old Generation）

2). 正确设置父类加载器

3). 保证findClass(String name)方法的逻辑正确性



**6、如何在运行时判断系统类加载器能加载哪些路径下的类？**

一是可以直接调用ClassLoader.getSystemClassLoader()或者其他方式获取到系统类加载器（系统类加载器和扩展类加载器本身都派生自URLClassLoader），调用URLClassLoader中的getURLs()方法可以获取到。二是可以直接通过获取系统属性java.class.path来查看当前类路径上的条目信息 ：System.getProperty(“java.class.path”)。如下所示，

``` java
public class Test {
	public static void main(String[] args) {
		System.out.println("Rico");
		Gson gson = new Gson();
		System.out.println(gson.getClass().getClassLoader());
		System.out.println(System.getProperty("java.class.path"));
	}
}/* Output: 
        Rico
		sun.misc.Launcher$AppClassLoader@6c68bcef
		I:\AlgorithmPractice\TestClassLoader\bin;I:\Java\jars\Gson\gson-2.3.1.jar
 *///:~ 
```



7、如何在运行时判断标准扩展类加载器能加载哪些路径下的类？

利用如下方式即可判断：

``` java
import java.net.URL;
import java.net.URLClassLoader;  

public class ClassLoaderTest {  
  
    /** 
     * @param args the command line arguments 
     */  
    public static void main(String[] args) {  
        try {  
            URL[] extURLs = ((URLClassLoader) ClassLoader.getSystemClassLoader().getParent()).getURLs();  
            for (int i = 0; i < extURLs.length; i++) {  
                System.out.println(extURLs[i]);  
            }  
        } catch (Exception e) {  
            //…  
        }  
    }  
} /* Output: 
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/access-bridge-64.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/dnsns.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/jaccess.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/localedata.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/sunec.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/sunjce_provider.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/sunmscapi.jar
        file:/C:/Program%20Files/Java/jdk1.7.0_79/jre/lib/ext/zipfs.jar
 *///:~ 

```


---
layout: post
title: 你应该知道的Java Classloader
categories: [技术]
tags: [Java]
---

## classloader是什么？

[ClassLoader Java API DOC](https://docs.oracle.com/javase/6/docs/api/java/lang/ClassLoader.html#findLoadedClass(java.lang.String))  

主要方法罗列在下面：

{% highlight java %}
package java.lang;

public abstract class ClassLoader {

  public Class loadClass(String name);
  protected Class defineClass(byte[] b);

  public URL getResource(String name);
  public Enumeration getResources(String name);
  
  public ClassLoader getParent()
}
{% endhighlight %}

当你第一次访问一个类并创建对象的时候，JVM需要先找到这个类的定义，并进行解析、链接、加载，生成该类的Class对象，再创建对象实例。ClassLoader的作用就是提供如何找到类和资源并加载（类class文件也是resource的一种）。以下是几个主要方法的功能：  

* loadClass  
传入类的全限定名，返回该类的Class实例

* defineClass  
传入类文件的字节流，返回类对象

* getResource  
传入资源名称，返回资源URL

我们可以以
> loadClass = defineClass(getResource(name).getBytes())

方式来理解类加载过程。


在JVM中，使用委托模型（delegation model）来寻找类和资源，应用可以通过继承Classloader的方式来动态调整加载行为。JVM定义了三类ClassLoader：

1. 引导类加载器（bootstrap class loader）
2. 扩展类加载器
3. 系统类加载器

其父子关系如下图：

![](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/image001.jpg)

如果没有重写改变加载行为，类会先交由父加载器进行加载，加载不成功再由自己进行加载。

## 参考链接

1. [深入浅出ClassLoader](http://ifeve.com/classloader/)
2. [Do you really get classloaders?](https://zeroturnaround.com/rebellabs/rebel-labs-tutorial-do-you-really-get-classloaders/)
3. [深入探讨Java类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)

## Key Point

* 可以通过继承Classloader来控制类加载的行为。

	1. 可以突破双亲委派模型（如OSGI[^osgi] ，Web容器的war包加载[^tomcat] 等）
	2. 可以从网络或其他地方加载类定义，只要提供类文件的字节（ defineClass）即可。

* JVM中确定一个类型的坐标是通过**类加载器和全类名**做到的。相同的类文件如果被不同的类加载器加载，两个类被认为类型不相同。

* 类加载器的如何使用？
  1. 直接引用使用，classloader.loadClass() 
  2. Class.forName(String name, boolean initialize, ClassLoader loader)
 
* 线程的上下文加载器  
  线程上下文类加载器（context classloader）并不是一类特殊的Classloader，**只是在java.lang.Thread中定义的一个Classloader引用**。   
  可以通过方法 getContextClassLoader() 和  setContextClassLoader(ClassLoader cl) 用来获取和设置线程的上下文类加载器。如果没有通过 setContextClassLoader(ClassLoader cl) 方法进行设置的话，线程将继承其父线程的上下文类加载器。  
  线程上下文加载器引用是为了解决SPI无法通过子加载器加载实现类的问题。比如JDBC的SPI接口定义，是在java.sql包下的，是通过Bootstrap Classloader加载的，但具体实现是在Classpath下，需通过系统加载器及其子类来完成。但双亲委派模型无法向子加载器委派，所以线程上下文加载器引用出现了，相当于提供了个向下引用的后门。
  
  
  {% highlight java %}
    //java/util/ServiceLoader.java:537
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
	{% endhighlight %}
	
* 在动态加载的条件下，那类到底是被哪个加载器加载的呢？后续会写篇专门的文章来解析。
	
* 常见的Classloader使用错误及定位方法
![](http://zeroturnaround.com/wp-content/uploads/2013/09/classloader-cheat-sheet.jpg)

* **慎用**classloader  
卸载Classloader时只要泄漏一个该加载器加载的对象，就非常容易发生内存泄漏。
所有的对象都有指向类对象的引用，所有的类对象都会有指向加载它的Classloader的引用。 而Classloader会包含所有他加载的类对象的引用，类对象里面又包含静态字段的引用。具体例子可以参考链接2.

![](https://zeroturnaround.com/wp-content/uploads/2013/08/classloader-reference-fields.jpg)

![](https://zeroturnaround.com/wp-content/uploads/2013/08/leak-classloader-counter-class.jpg)

* 动态加载怎么做？  
通过新的Classloader生成新的对象并复制老对象的状态，新对象的修改就可以生效了。
下图展示了Servlet容器reload的机制。其中通过序列化、反序列化的方式来快速复制对象状态。

![](https://zeroturnaround.com/wp-content/uploads/2013/08/serialize-deserialize.jpg)

## 引用
[^tomcat]: https://tomcat.apache.org/tomcat-7.0-doc/class-loader-howto.html
[^osgi]: https://www.ibm.com/developerworks/cn/java/j-lo-osgi/


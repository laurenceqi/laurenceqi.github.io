---
layout: post
title: Double-checked Lock 到底有什么问题？
categories: [技术]
tags: [Java]
---

为了减少同步的开销，又想lazy initialize实例，所以一种”聪明“的做法出现了：

{% highlight java linenos %}
// double-checked-locking - don't do this!

private static Something instance = null;

public Something getInstance() {
  if (instance == null) {
    synchronized (this) {
      if (instance == null)
        instance = new Something();
    }
  }
  return instance;
}
{% endhighlight %}

但是，**这并不总是正确工作**。

### 原因一
当线程 A 执行到第 9 行时，会执行实例的创建、初始化及赋值，如果此时线程 B 执行到第 6 行，读取 instance 对象的值，因为该步骤不存在任何 happens before 关系， 所以可能读到指令重排序后的值。如果重排序后，指令顺序是：

1. 线程 A 创建 Something 对象实例 S
3. 线程 A 将 S 的引用赋值给 instance 对象
4. 线程 B 读取 instance 对象不为 null， 并返回引用
5. 线程 A 初始化instance对象

该种情况下，就会返回一个没有初始化的实例，导致程序运行错误。

重排序后的字节码可能是这样：

{% highlight assembly %}
0206106A   mov         eax,0F97E78h
0206106F   call        01F6B210                  ; allocate space for
                                                 ; Singleton, return result in eax
02061074   mov         dword ptr [ebp],eax       ; EBP is &singletons[i].reference 
                                                 ; store the unconstructed object here.
02061077   mov         ecx,dword ptr [eax]       ; dereference the handle to
                                                 ; get the raw pointer
02061079   mov         dword ptr [ecx],100h      ; Next 4 lines are
0206107F   mov         dword ptr [ecx+4],200h    ; Singleton's inlined constructor
02061086   mov         dword ptr [ecx+8],400h
0206108D   mov         dword ptr [ecx+0Ch],0F84030h
{% endhighlight %}

&singletons[i].reference 对引用的赋值早于构造函数。

### 原因二
即使上述步骤中第6行读到了实例变量都赋值的实例 S， S的引用类型实例变量所指向的对象有可能也是状态不对的。所以返回的实例仍然存在问题。比如:

{% highlight java %}
class Something {
	Something  a,
	Something1 b,
}
{% endhighlight %}

即使实例S的实例变量a != null, b != null, a,b所指向的对象仍有可能是未初始化完全的。

### 正确做法
在老的Java内存模型定义下，并没有好的办法来解决Double checked问题，但可以通过 On Demand Holder 类来解决。

{% highlight java %}
private static class LazySomethingHolder {
  public static Something something = new Something();
}

public static Something getInstance() {
  return LazySomethingHolder.something;
}
{% endhighlight %}

在新的内存模型定义下，则可以通过 Volatile 关键字来解决。

{% highlight java %}

private static volatile Something instance = null;

public static Something getInstance() {
  if (instance == null) {
    synchronized (this) {
      if (instance == null)
        instance = new Something();
    }
  }
  return instance;
}
{% endhighlight %}

为什么使用 volatile 关键词就可以解决该问题呢？

### 引用

1.  [http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
2.  [https://www.javaworld.com/article/2074979/java-concurrency/double-checked-locking--clever--but-broken.html?page=2](https://www.javaworld.com/article/2074979/java-concurrency/double-checked-locking--clever--but-broken.html?page=2)
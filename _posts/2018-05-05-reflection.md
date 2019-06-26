---
layout:     post
title:      Java反射
subtitle:   
date:       2018-05-05
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

反射可以使程序在运行时动态地加载需要的类，并且在运行期间可以获得任意对象的类型并调用其方法，而不需要事先（主要是编码和编译中）知道运行对象是谁。

# 使用反射

每一个类都对应一个class文件，这些class文件会在需要的时候被虚拟机动态地加载，并在内存中生成一个代表这个类的java.lang.Class对象，可以通过Class的静态方法forName根据字符串获得该Class对象的引用或者直接获取一个对象的class(例如String.class)，再或者，可以通过调用某个对象的getClass方法获得Class对象。如果查看Class类的源码可知，forname方法实际上调用native方法forname0，再接下来的过程就涉及到ClassLoader的加载过程，就不详细叙述了。获得了Class的对象引用，接下来就可以根据这个引用来创建对象的实例了，主要有两种方式：使用Class对象的newInstance()方法来创建Class对象对应类的实例，以及通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。

```java
//方法1
Class<?> c = String.class;
Object str = c.newInstance();

//方法2
Class c = String.class;
Constructor constructor = c.getConstructor(String.class);
Object obj = constructor.newInstance("23333");

```

获得了对象实例，接下来就可以根据需要来进行想要的操作了，比如获取并调用方法：

```java
//前面获取Class的引用并实例化对象
//注意这个int.class和Integer.class不同，基本类型的class都是不存在的
//int.class对应的Class对象是JVM合成出来的，并不是从Class文件中加载出来的，纯粹属于JVM的实现细节
Method m = c.getMethod("add",int.class,int.class);
Object result = m.invoke(o,1,4)
```

同样也可以获得对象的成员变量，getField获得公有成员变量，getDeclaredField获得所有已声明的成员变量（除了从父类继承的）。

利用反射创建数组有点不同：

```java
Object array = Array.newInstance(c,32);
Array.set(array,0,1);
```

Array是java.lang.reflect下的Array类（注意与util包下的Arrays区分），创建的数组仍旧是一个Object对象而不是真正的数组。

# 反射的缺陷

+ 产生不少临时对象，运行效率不高。
+ 验证等防御代码过于繁琐，计算消耗大量时间。
+ 因为反射涉及到动态加载，JIT无法对这一过程做优化。

# 反射的优化

1. 利用缓存，可以先把Class对象，Object对象，Method对象等缓存，比如存到HashMap里，这样再需要forName查找Class对象以及创建实例时就可以从缓存中取了。
2. 使用字节码生成技术，比如使用ASM框架，它能被用来动态生成类或者增强既有类的功能。ASM可以直接产生二进制class文件，也可以在类被加载入Java虚拟机之前动态改变类行为。ASM从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。

# 参考文章

[深入解析Java反射(1)-基础](http://www.sczyh30.com/posts/Java/java-reflection-1/#%E4%B8%80%E3%80%81%E5%9B%9E%E9%A1%BE%EF%BC%9A%E4%BB%80%E4%B9%88%E6%98%AF%E5%8F%8D%E5%B0%84%EF%BC%9F)
[Java反射在JVM的实现](http://www.importnew.com/21211.html)

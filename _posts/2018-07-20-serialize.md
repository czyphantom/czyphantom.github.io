---
layout:     post
title:      Java序列化
subtitle:   
date:       2018-07-20
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

由于只有在JVM运行时，对象才能存在，当JVM停止时，内存中的对象便不可使用。实际中可能会在JVM停止后仍能保存特定的对象，以便重新读取该对象，这时候就需要用到Java的序列化。序列化保存的是对象的状态，即成员变量的值，静态变量则不考虑在内。实现序列化和反序列化需要实现Serializable接口，如果想要同时序列化一个类的父类，那么需要父类也实现Serializable接口。如果不想要某个变量被序列化，可以在这个变量前加上transient关键字，可以阻止变量被序列化（反序列化时，transient变量被设置为初始值0）。例：

```java
public class test implements Serializable{
    private static final long serialVersionUID = -7207636737744804775L;
    private String a;
    private transient  int b;

    public String getA() {
        return a;
    }

    public int getB() {
        return b;
    }

    public void setB(int b) {
        this.b = b;
    }

    public void setA(String b) {
        this.a = b;
    }
}

public class a {
    public static void main(String[] args) {
        test t = new test();
        t.setA("abc");
        t.setB(5);
        ObjectOutputStream out = null;
        try {
            out = new ObjectOutputStream(new FileOutputStream("1.txt"));
            out.writeObject(t);
        } catch(Exception e) {
            e.printStackTrace();
        }

        File file = new File("1.txt");
        ObjectInputStream in = null;
        try {
            in = new ObjectInputStream(new FileInputStream(file));
            test obj = (test)in.readObject();
            System.out.println(obj.getA());
            System.out.println(obj.getB());
        }catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```


运行结果为:

abc
0

可以看到在实现Serializable接口的类中，有一个private static final long类型的变量serialVersionUID，此即序列化ID，通常使用1L或一个随机的不重复的数。如果没有指定序列化ID，Java编译器会自动给class通过摘要算法生成一个，此时如果在文件中多一个空格或者少一个空格都会使产生的序列化ID不同。因此如果需要在对象中添加或者删除变量，需要指定序列化ID。序列化ID主要让JVM判定是否允许反序列化，只有同一次编译的所产生的序列化ID是相同的，如果序列化ID不同，那么JVM拒绝加载该类。

在序列化过程中，如果被序列化的类中定义了writeObject和readObject方法，在对ObjectOutputStream或ObjectInputStream调用writeObject和readObject方法时，虚拟机会试图调用对象类里的writeObject和readObject方法，进行用户自定义的序列化和反序列化。如果没有这两个方法，则默认调用是 ObjectOutputStream的defaultWriteObject方法以及ObjectInputStream的defaultReadObject方法。以ObjectOutputStream为例，writeObject调用栈如下：

writeObject ---> writeObject0 --->writeOrdinaryObject--->writeSerialData--->invokeWriteObject(或者defaultWriteFields)

最后一步即是通过反射调用自定义的writeObject方法。

Java序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件的为同一对象时，并不会再将对象的内容进行存储，而只是再次存储一份引用。如果在一次写之后，对对象再进行修改，再写入文件，那么只会保存第一次的结果，因为知道第二次和第一次引用的对象是相同的，需要注意。


参考文章

[Java 序列化的高级认识](https://www.ibm.com/developerworks/cn/java/j-lo-serial/)
---
layout:     post
title:      Java注解
subtitle:   
date:       2018-03-13
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

# 简介

注解自Java5开始引入，提供了一种安全的类似注释的机制，用来将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联。为程序的元素（类、方法、成员变量）加上更直观更明了的说明，这些说明信息是与程序的业务逻辑无关，并且供指定的工具或框架使用。注解像一种修饰符一样，应用于包、类型、构造方法、方法、成员变量、参数及本地变量的声明语句中。

Java注解是附加在代码中的一些元信息，用于一些工具在编译、运行时进行解析和使用，起到说明、配置的功能。注解不会也不能影响代码的实际逻辑，仅仅起到辅助性的作用。包含在java.lang.annotation包中。

注解的作用：

1. 生成文档。
2. 实现配置文件的功能，减少复杂的配置文件使用。
3. 编译前进行格式检查，比如@Override注解。

**注解本质是一个实现了Annotation的特殊接口，具体实现类是Java运行时生成的动态代理类。**通过反射获取注解时，返回的是Java运行时生成的动态代理对象。

# 元注解

在java.lang.annontation包下有4个元注解，用来注解其他注解：

1. @Retention – 定义该注解的生命周期，分别可以定义编译阶段丢弃，类加载阶段丢弃，始终不丢弃。
2. @Target – 表示该注解用于什么地方。默认值为任何元素，表示该注解用于什么地方。
3. @Documented – 一个简单的Annotations标记注解，表示是否将注解信息添加在java文档中。
4. @Inherited – 定义该注释和子类的关系。@Inherited元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

# 自定义注解

注解需要定义为@interface，且不能实现其他接口或者继承其他类，参数成员只能用public或者默认两个访问修饰符，参数成员只能使用基本类型和String，Enum，Class和Annontations几种类型或数组。

注解通常需要使用注解处理器来处理注解，示例如下：

```java
/**
* 定义注解
*/

@Retention(RetentionPolicy.RUNTIME)
public @interface Name {
    public String name() default "xxx";
}

/**
* 注解使用
*/

public class Test {
    @Name(name="czy")
    private String name;
}
/**
* 注解处理器，比如打印得到的注解值，也可以做其他操作。
*/

public class NameUtil {
    public static void getName(Class<?> clazz) {
        Field[] fields = clazz.getDeclaredFields();
        for(Field field :fields) {
            if(field.isAnnotationPresent(Name.class) {
                Name name = (Name) field.getAnnotation(Name.class);
                String  myName ="名："+name.name();
                System.out.println(myName);
            }
        }
    }
}
```

# 总结

简单来说，可以把注解理解为用来解释Java代码的部分，一般可以用来缩短代码量或者减少配置文件的使用，不过由于处理注解时使用反射，因此可能比较慢。

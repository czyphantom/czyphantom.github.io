---
layout:     post
title:      Java动态类型
subtitle:   
date:       2019-05-07
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java
---

Java自JDK1.7以后开始支持动态类型语言支持。

# 动态类型语言和静态类型语言

动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期进行的，而编译期就进行类型检查过程的语言就是静态语言，比如Java。简单看一个例子：

```java
obj.get(1);
```

如果声明obj是List类型，那么它的值就必须是List的实现类才是合法的，如果不是，即使obj确实有一个get方法，代码依旧不能运行。但是在动态语言中，只要obj确实有这个方法，那方法调用就可以成功。出现这种差异主要是因为Java语言在编译期间却已将get(int)方法完整的符号引用生成出来，作为方法调用指令的参数存储到Class文件中。

这个符号引用包含了此方法定义在哪个具体类型之中、方法的名字以及参数顺序、参数类型和方法返回值等信息，通过这个符号引用，虚拟机就可以翻译出这个方法的直接引用。而在动态类型语言中变量obj本身是没有类型的，变量obj的值才具有的类型，编译时候最多只能确定方法名称、参数、返回值这些信息，而不会去确定方法所在的具体类型。

# invokedynamic指令和java.lang.invoke的出现

invokedynamic指令主要用于JDK1.8的lambda的实现，在JDK1.7中并没有办法生成invokedynamic指令，每一处含有invokedynamic指令的位置都被称作“动态调用点（Dynamic Call Site）”，这条指令的第一个参数不再是代表方法符号引用的CONSTANT_Methodref_info常量，而是变为JDK1.7新加入的CONSTANT_InvokeDynamic_info常量，从这个新常量中可以得到3项信息：引导方法（Bootstrap Method，此方法存放在新增的BootstrapMethods属性中）、方法类型（MethodType）和名称。引导方法是有固定的参数，并且返回值是java.lang.invoke.CallSite对象，这个代表真正要执行的目标方法调用。根据CONSTANT_InvokeDynamic_info常量中提供的信息，虚拟机可以找到并且执行引导方法，从而获得一个CallSite对象，最终调用要执行的目标方法上。之后再学lambda表达式的时候再具体分析，这里只讲以下java.lang.invoke包。

java.lang.invoke这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这条路之外，提供一种新的动态确定目标方法的机制，称为Method Handle，即方法句柄。对于一个方法句柄来说，它的类型完全由它的参数类型和返回值类型来确定，而与它所引用的底层方法的名称和所在的类没有关系。

来看一个invoke的简单使用：

```java
public class DynamicTest {
    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }
    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
        //无论obj最终是哪个实现类，下面这句都能正确调用到println方法。
        getPrintlnMH(obj).invokeExact("abc");
    }
    private static MethodHandle getPrintlnMH(Object reveiver) throws Throwable {
        //MethodType：代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）和具体参数（methodType()第二个及以后的参数）。
        MethodType mt = MethodType.methodType(void.class, String.class);
        //lookup()方法来自于MethodHandles.lookup，这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
        //因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即是this指向的对象，这个参数以前是放在参数列表中进行传递，
        //现在提供了bindTo()方法来完成这件事情。
        return lookup().findVirtual(reveiver.getClass(), "println", mt).bindTo(reveiver);
    }
}
```

方法getPrintlnMH()中实际上是模拟了invokevirtual指令的执行过程，只不过它的分派逻辑并非固化在Class文件的字节码上的，而是通过一个具体方法来实现。而这个方法本身的返回值（MethodHandle对象），可以视为对最终调用方法的一个“引用”。

MethodHandle的使用方法和效果上与Reflection都有众多相似之处,但是也有不同：

+ 反射和方法句柄机制本质上都是在模拟方法调用，但是反射是在模拟Java代码层次的方法调用，而方法句柄是在模拟字节码层次的方法调用。在MethodHandles.Lookup上的三个方法findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual & invokeinterface和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用反射API时是不需要关心的。
+ 反射中的java.lang.reflect.Method对象远比方法句柄机制中的java.lang.invoke.MethodHandle对象所包含的信息来得多。前者是方法在Java一端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含有执行权限等的运行期信息。而后者仅仅包含着与执行该方法相关的信息。用开发人员通俗的话来讲，反射是重量级，而方法句柄是轻量级。
+ 由于方法句柄是对字节码的方法指令调用的模拟，那理论上虚拟机在这方面做的各种优化（如方法内联），在方法句柄上也应当可以采用类似思路去支持。而通过反射去调用方法则不行。

# 参考文章

[解析JDK 7的动态类型语言支持](http://www.infoq.com/cn/articles/jdk-dynamically-typed-language)

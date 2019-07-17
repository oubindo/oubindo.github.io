---
title: 你所不知道的Method Handles
date: 2018-09-30 10:15:00
tags: Android进阶
categories: android
---

## Method Handles
Method Hanldes是在Java 7引入的概念。全限定名是java.lang.invoke.MethodHandles。在这篇文章中，我们将学会如何创建，使用MethodHandles及它的原理。

### 1.介绍
Method Handles的引入是为了与已经存在的java.lang.reflect API相配合。他们分别是为了解决不同的问题而出现的。从性能角度上说，MethodHandle api要比反射快很多因为访问检查在创建的时候就已经完成了，而不是像反射一样等到运行时候才检查。但同时，Method Handles比反射更难用，因为没有列举类中成员，获取属性访问标志之类的机制。
另外，MethodHandles可以操作方法，更改方法参数的类型和他们的顺序。而反射则没有这些功能。
从以上角度看，反射更通用，但是安全性更差，因为可以在不授权的情况下使用反射对象。而method Handles遵从了分享者的能力。所以method handle是一种更低级的发现，适配和调用方法的方式，唯一的优点就是更快。所以反射更适合主流Java开发者，而method handle更适用于对编译和运行性能有要求的人。


### 2.使用
1.要使用method handle，首先需要得到Lookup。这是创造方法，构造函数，属性的method handles的工厂类。
```java
// public方法的Lookup
MethodHandles.Lookup publicLookup = MethodHandles.publicLookup();
// 所有方法的Lookup
MethodHandles.Lookup lookup = MethodHandles.lookup();
```

2.要创建MethodHandle，lookup需要一个定义了它的类型的MethodType对象。这里的类型包括了传入参数的类型，和最后返回的类型，要一一对应。第一个是返回类型，如果没有返回值就是Void.class, 后面是可变的传入参数的类型。
>  a MethodType represents the arguments and return type accepted and returned by a method handle or passed and expected by a method handle caller.

例如
```java
// 接收数组，返回一个List对象
MethodType mt = MethodType.methodType(List.class, Object[].class);
```

3.查找MethodHandle
Lookup之所以叫Lookup自然是因为他们有查找MethodHandle的能力。先看看他的方法。
![屏幕快照 2018-10-19 上午10.19.20.png](http://ovkwd4vse.bkt.clouddn.com/屏幕快照 2018-10-19 上午10.19.20.png)

4.接下来就可以进行查找并调用了
```java
MethodType mt = MethodType.methodType(String.class, char.class, char.class);
MethodHandle replaceMH = publicLookup.findVirtual(String.class, "replace", mt);
 
String output = (String) replaceMH.invoke("jovo", Character.valueOf('o'), 'a');
```

5.方法调用细则：
有三种方法可以调用方法invoke(), invokeWithArugments()和invokeExact()，当我们使用invoke时，我们必须固定arguments的数目。
```java
// invoke使用
MethodType mt = MethodType.methodType(String.class, char.class, char.class);
MethodHandle replaceMH = publicLookup.findVirtual(String.class, "replace", mt);
String output = (String) replaceMH.invoke("jovo", Character.valueOf('o'), 'a');

// invokeWithArguments使用
MethodType mt = MethodType.methodType(List.class, Object[].class);
MethodHandle asList = publicLookup.findStatic(Arrays.class, "asList", mt);
List<Integer> list = (List<Integer>) asList.invokeWithArguments(1,2);

// invokeExact
MethodType mt = MethodType.methodType(int.class, int.class, int.class);
MethodHandle sumMH = lookup.findStatic(Integer.class, "sum", mt);
int sum = (int) sumMH.invokeExact(1, 11);
```
具体的区别是：
> 与invokeExact方法不同，invoke方法允许更加松散的调用方式。它会尝试在调用的时候进行返回值和参数类型的转换工作。这是通过MethodHandle类的asType方法来完成的，asType方法的作用是把当前方法句柄适配到新的MethodType上面，并产生一个新的方法句柄。当方法句柄在调用时的类型与其声明的类型完全一致的时候，调用invoke方法等于调用invokeExact方法；否则，invoke方法会先调用asType方法来尝试适配到调用时的类型。如果适配成功，则可以继续调用。否则会抛出相关的异常。这种灵活的适配机制，使invoke方法成为在绝大多数情况下都应该使用的方法句柄调用方式。



参考：
https://www.baeldung.com/java-method-handles
https://www.cnblogs.com/night-wind/p/4405564.html
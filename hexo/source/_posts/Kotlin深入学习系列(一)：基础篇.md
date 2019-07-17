---
title: Kotlin深入学习系列(一):基础篇
date: 2018-04-08 20:08:01
tags: Kotlin
categories: Kotlin
---

# 基础篇
## 基本类型
1.数字  
数字与java中基本相同，主要是下面几点不同：
- kotlin对于数字没有隐式扩展转换(如从int转为long)
- 在kotlin中字符不是数字

2.表示方式
kotlin会把数字存储为jvm的原生类型，但是只有在**需要可空引用或泛型时**数字会被装箱。  
但是这里存在着一个坑：当取值在[-128, 127]之间时不会被装箱，而在这之外的int数字才会被装箱，如下
```
    val b:Int = 5    // 被编译为 short b = 5;
    val b1:Int? = b  // 被编译为 Integer i = Integer.valueOf(5)
    val b11:Int? = b  
    println(b1 === b11) // 注意这里打印的是 'true'  

    val a: Int = 10000  
    val a1: Int? = a  
    val a11: Int? = a  
    println (a1 === a11 ) //注意这里打印的是 'false'  
```

这是因为JVM把[-128,127]的所有int数字的装箱全部缓存了，任何指向这个范围的对象，都不可能被另外"创建"，应该是为了缓存最常用的数字以节省性能吧。

3.在kotlin中万物皆对象，=== 是比较对象之间的地址是否相等， == 是比较对象的大小是否相等。

4.数据类型转换：需要使用toByte(),toInt之类的转换。

5.浮点数比较有些很棒的特性：
- 区间实例以及区间检测：a..b、 x in a..b、 x !in a..b。 这里可以使用两个点来表示，甚至比python更方便。
- NaN与其自身相等，而且比任何其他元素都大

6.字符Char不能直接当做数字，用单引号，与java一样。并且当需要可空引用时也会被装箱。

7.数组：数组直接用Array<T>表示
```java
// 一.已知所有元素
val a = arrayOf(1,2,3) // array[1,2,3]
val b = Array(5, { i -> (i * i).toString() })  // 数组大小和一个Array构造函数
// 二.空数组
val c = emptyArray()/ arrayOfNulls<String>(0)  // 创建一个长度为0的null数组
// 三.基本类型
val d = intArrayOf(1,2,3)  // 对于基本类型数组有特殊优化，但是得到的IntArray并不是Array的子类。
// 四.创建一个已知数量的数组
val a = arrayOfNulls<String>(5)
```

kotlin中数组是invariant的，不能把Array<String>赋值给Array<Any>,但是可以是Array<out Any>

8.字符串模板  
字符串可以包含模板表达式 ，即一些小段代码，会求值并把结果合并到字符串中。模板表达式以美元符（$）开头，由一个简单的名字构成。如果要直接输出$符，只要不加变量即可:
```java
val s = "abc"
println("$s.length is ${s.length}") // 输出“abc.length is 3”
print("$8.8")
```

## 控制流
1.if语句表达三元运算符：val max = if (a > b) a else b。与python中的表达十分相似：max = a if a > b else b

2.When表达式  
when是一个加强的switch语句，可以作为语句也可以作为表达式。可以替代if else if链
```
fun hasPrefix(x: Any) = when(x) {
    1,2 -> true
    is String -> x.startsWith("prefix")
    else -> false
}
```

3.for可以循环任何提供了迭代器的对象。

4.Kotlin中任何表达式都可以用标签(label)来标记。标签的格式为标识符后跟@符号
```
loop@ for (i in 1..100) {
    for (j in 1..100) {
        if (……) break@loop
    }
}
```
例子中的break可以直接跳出两重循环。同样的return,continue都可以使用这种操作。

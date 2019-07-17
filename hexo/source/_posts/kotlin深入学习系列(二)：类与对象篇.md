---
title: Kotlin深入学习系列(二):OOP篇
date: 2018-04-10 13:38:32
tags: Kotlin
categories: Kotlin
---

# 类与对象
## 一.类
### 构造函数  
在 Kotlin 中的一个类可以有一个主构造函数和一个或多个次构造函数。主构造函数是类头的一部分：它跟在类名（和可选的类型参数）后。其中由于主构造函数内不能写代码，主要的初始化代码需要在 init{} 中书写。在实例初始化期间，初始化块按照它们出现在类体中的顺序执行，与属性初始化器交织在一起。也就是说这里的**初始化块init其实就是被当成了属性初始化**。主构造函数中的参数默认为类参数使用。

次构造函数可以通过constructor(param)来定义，**所有的构造函数都必须委托给主构造函数，可以直接委托或者通过别的次构造函数简洁委托。** 委托方法是在constructor后面加个this()。

但是这里又有一个坑，次构造函数不能使用val或者var！例子如下：
```java
class Pair<K, V>(val first: K, val second: V) {

    init{
        println("init")
        println(first)
        println(second)
    }

    constructor(val first: K): this(first, 0){    // 这里的val报错。
        println("constructor")
        println(first)
        println(second)

    }
}
```


### 继承  
Kotlin所有类都有一个超类Any。声明类时如果使用了open来修饰，那就可以被继承，否则默认不允许继承。

继承的时候如果子类有主构造函数，那么必须用父类的主构造函数参数初始化。否则必须有次构造函数调用super来调用父类的构造函数。
```java
class HeritLearning: ClassLearning{
    constructor(name: String) : super(name){

    }

}

// 或者
class HeritLearning(name: String) : ClassLearning(name){

}
```

如果一个方法同时和多个超类的相同，那就在调用父类方法的时候使用super<父类>.func()来区分父类。


### 覆盖属性或方法
开放覆盖的属性和方法必须用open修饰，覆盖的必须用override修饰。覆盖属性的时候var可以覆盖val，反之则不行。

### 接口
接口默认为open，不需要主构造函数。


## 二.字段
字段的完整语法：
```java
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```
也就是是说，当我们没有声明getter和setter时默认就已经有这些东西了。当我们声明以后可以得到自定义的实现。如果想屏蔽getter或者setter，可以使用private set。
```java
var setterVisibility: String = "abc"
    private set // 此 setter 是私有的并且有默认实现
```
在访问器里面，我们可以使用field来代替此字段
```java
var counter = 0 // 注意：这个初始器直接为幕后字段赋值
    set(value) {
        if (value >= 0) field = value
    }
```

### lateinit
属性初始化要点如下：
- 当我们在类外面声明属性为可空或非空时，都必须进行初始化
- 声明为lateinit的对象必须是可空对象，非空对象不行。
- 使用.isInitialized来判断属性是否被初始化了。

![](http://ovkwd4vse.bkt.clouddn.com/markdown-img-paste-20180410161250423.png)


## 三.扩展
所有扩展的定义域都在包里，使用之外的扩展就需要导入。

1.扩展函数
kotlin可以通过扩展来扩展类的新功能而无需继承或装饰。原理就是：
```java
fun ClassLearning.f(str: String){
    this.arr[3] = 0
}

open class ClassLearning(val name: String, val age: Int){
    var arr = Array<Int>(5, {1;2;4;2;1})
}
```
反编译之后如下：
```java
public final class ClassLearningKt {
   public static final void f(@NotNull ClassLearning $receiver, @NotNull String str) {
      Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
      Intrinsics.checkParameterIsNotNull(str, "str");
      $receiver.getArr()[3] = 0;
   }

   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      ClassLearning l = new ClassLearning("oubin");
      f(l, "f方法");
   }
}
```
可以看到被扩展的对象被打包成了receiver，然后就是这个对象直接操作他自身的元素了。也就是说，扩展对象并没有改变他们所扩展的类，而是通过生成另外一个方法来操作。所以我们可以发现，这个方法绝对是编译器生成的，所以对于运行期重载的情况怕是用不了了。
```
open class C

class D: C()

fun C.foo() = "c"

fun D.foo() = "d"

fun printFoo(c: C) {
    println(c.foo())
}

printFoo(D())  // 会print c
```


2.扩展属性  
扩展属性不能有初始化器，他们的行为只能由显式的getter和setter定义。扩展属性也是在新类中生成了一个get方法，然后将扩展属性的get()返回的值进行返回。

3.伴生对象扩展：同样伴生对象也可以定义扩展函数和属性。


## 四.数据类
```
data class Person(val name: String){
    var age: Int = 0
}

fun Person.copy(name:String = this.name) = Person(name)
```
数据类可以将部分属性放入括号内，从而排除这些属性。进行copy的时候可以如上定义，然后Person对象可以直接调用copy函数

数据类的解构声明：
```
val jane = User("Jane", 35)
val (name, age) = jane
println("$name, $age years of age")
```

他是使用了利用了componentN()函数。
![](http://ovkwd4vse.bkt.clouddn.com/markdown-img-paste-20180410170358854.png)


## 五.嵌套类
内部类通过标记为inner可以访问外部类的成员，因为会带有一个对外部类的对象的引用。

匿名内部类可以使用对象表达式来实现。

## 六.对象
1.对象表达式: 样式为 object: X(){...}  
当我们不需要任何超类型时，就写做 val a = object{ val x = 3}, 这样我们就可以访问a.x
它的实现原理是通过new Object()去生成了这样一个对象并定义相关属性。

对象表达式可以访问外部的属性，并且比java匿名内部类更牛的是它不限制是final对象。


2.伴生对象
当我们在类中定义伴生对象时发生了什么：
```java
class HeritLearning(name: String) : ClassLearning(name){
    companion object {
        val x: String = "ds"

        fun get(){
            println()
        }
    }
}

// 反编译后为
public static final class Companion {
      @NotNull
      public final String getX() {
         return HeritLearning.x;
      }

      public final void get() {
         System.out.println();
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
  }
```
可以看到我们的companion被编译成了一个静态内部类，属性被定义为外部类的静态属性，而方法确是这个静态内部类的方法。由此，我们知道伴生方法是访问不了非伴生属性的。


## 七.委托
```java
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

class Derived(b: Base) : Base by b

fun main(args: Array<String>) {
    val b = BaseImpl(10)
    Derived(b).print() // 输出 10
}

```



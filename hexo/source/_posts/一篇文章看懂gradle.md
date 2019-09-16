---
title: 一篇文章基本看懂gradle
date: 2019-08-30 10:09:44
tags: gradle
---


# 一篇文章基本看懂gradle
Hi，大家好啊。笨鸟之旅已经很久都没有更新了，感谢大家这么久以来还把我留在列表里。这么久以来不更新的原因主要是我本人进入了一个迷茫期，对于工作和生活都提不起兴致来。加上每天工作的时间太长，一回家就瘫在床上了。这也导致了很久之前的文章计划一直都搁浅，做的计划一次一次的delay。于是生活进入了一个负循环。但是生活还是要向上看，还是要坚持去努力做点事情，努力去变得更好，所以我又来了。以后一定会努力的学习，努力的发文。

这段时间来学习了gradle，也体会到了gradle从初步理解到基本熟悉，再到深入源码这样一个过程中的一些曲折。这篇文章主要是gradle的基础知识篇。看完这篇文章，你可以：
- 清楚gradle的定义和解决的痛点
- 基本理解Android gradle的运作机制
- 基本理解gradle的大部分语法
- 学会基本的groovy开发

如果你想关注gradle更深入的一些知识，请继续关注后续gradle文章。

## what is gradle?
先来看一段维基百科上对于gradle的解释。
> Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化构建工具。它使用一种基于Groovy的特定领域语言来声明项目设置，而不是传统的XML。当前其支持的语言限于Java、Groovy和Scala，计划未来将支持更多的语言。

可能刚接触gradle的同学都不是很了解gradle的这个定义。可能就只会跟着网上的教程copy一点配置，但是不理解这些配置背后的原理。那么怎么来理解这句话呢，我们可以把握到三个要点：首先，它是一种`构建工具`，其次，gradle是`基于maven概念`的，最后，使用`groovy`这种语言来声明。要理解这几句话，我们先考虑几个场景。

1.`渠道管理`：国内手机市场有大大小小数十个，大的手机厂商也有五六个，每个厂商可能又有不同的定制rom。如果我们要为不同市场和厂商进行适配，那就需要写这样的代码
```
if(isHuawei) {
    // dosomething
} else if(isOppo) {
    // dosomething
}
```
这样的话，繁琐不说，对单个手机而言大量的无用代码被编译进apk中，包体积和运行速度都会受影响。为了解决这个问题，gradle引进了productFlavor和buildType的能力，能根据情况来进行打包。所以说他是一个`自动化构建工具`。可以看[官方文档](https://developer.android.com/studio/build?hl=zh-cn)

2.`依赖管理`：我们通常会在项目中引入各种三方库进行代码复用。比如，直接手动把jar或者aar copy到项目中，然后添加依赖。这种方法缺陷很明显，首先配置和删除流程很繁琐，其次，同一个jar可能会被多个项目所引用，导致不知不觉就copy了多个jar。最后，版本管理艰难。为了解决这个问题，gradle是基于maven仓库，配置和删除的时候仅需要对仓库的坐标进行操作，所有的库都会被gradle统一管理，大多数情况下每个库只会有一个版本存在于项目中，并且每个库只会有一个副本存在于项目中。

所以gradle其实不是什么神秘的东西，只是基于某种语言(groovy, java, kotlin)的一种构建工具而已。只要我们大概掌握了基本的用法和他的内部原理，日常工作中就会知道自己网上搜到的命令是什么意思啦。skr~


## 小试牛刀-android中的gradle
![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/20190829093346.png)  

咱们先看看日常工作中经常用到的几个gradle文件。可以看到主要有有三个文件：
1.build.gradle
根文件下放的通常放的是针对整个工程的通用配置，每个module下面的build.gradle文件是针对每个module自身的配置。
```groovy
buildscript {
    ext.kotlin_version = '1.2.71'
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.2.1'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```
这是一个默认的配置，我们可以看到有buildscript，allprojects，repositories，dependencies几个配置项，这些配置项是干嘛的呢，很多的同学在刚学gradle的时候都是一脸懵逼的。这些其实是gradle的一种特定的语法，我们称之为DSL(domain-specific language)。可以参考[官网](https://docs.gradle.org/current/dsl/index.html)。这里可以看到allprojects代理的是每个project，可以理解成我们的每个module，也就是对我们所写的每个module的配置。buildscript主要配置的是打包相关的东西，比如gradle版本，gradle插件版本等，这些都是针对构建工具自己的配置。repositories，dependencies是三方库的仓库和坐标。所以根目录的build.gradle相当于是整体的配置。

而module下的build.gradle主要是android，dependencies等配置项。
```
apply plugin: 'com.android.application'

android{
    ...
}
dependencies{
    ...
}
```
可能有些同学会感到奇怪，为啥我们在[官网](https://docs.gradle.org/current/dsl/index.html)没有看到android这个配置项呢？这个主要是因为它并不是gradle的DSL，某种意义上说应该算是android特有的，是通过Android的插件'com.android.application'带进来的配置项。我们如果把第一行删掉，就会发现android{}这个配置项找不到了。

所以，我们可以发现，build.gradle里面的配置项，要么是gradle自带的，要么是各种插件定义的。有不认识的配置项，就去官网查询一下就好了，授人以鱼不如授人以渔嘛。我们后面也会讲解到引进插件的方式和怎么定义插件和配置项。


2.settings.gradle
这个文件主要是决定每个module是否参与构建。我们可以这样去理解，settings.gradle相当于是每个module的开关，关上了这个module就不能使用了，别的依赖到它的module也都会出问题。

3.gradle.properties
这里主要是增加和修改一些可以在构建过程中直接使用的参数。不只是可以添加自定义参数，还可以修改系统的参数哦~

总结一下，就是说根目录下有一个build.gradle，处理整个工程的配置项，根目录下的settings.gradle配置整个工程中参与构建的module，每个module自己有一个build.gradle，处理自己模块的配置。这就是android构建的一个大概情况。当然，看了这一部分肯定还是不懂怎么去写的，接下来我们走进代码层面。


## groovy-学gradle的密钥
gradle可以使用groovy，kotlin，java等语言进行书写，但是groovy相对来说是目前比较流行的gradle配置方式，下面我们讲解一点groovy基础。不讲太多，够用就行。

1.字符串

groovy的字符串分为两种java.lang.String和groovy.lang.GString。其中单引号和三引号是String类型的。双引号是GString类型的。支持占位插值操作。和kotlin一样，groovy的插值操作也是用`${}`或者`$`来标示，`${}`用于一般替代字串或者表达式，`$`主要用于A.B的形式中。

```groovy
def number = 1 
def eagerGString = "value == ${number}"
def lazyGString = "value == ${ -> number }"

println eagerGString
println lazyGString

number = 2 
println eagerGString 
println lazyGString
```


2.字符Character

Groovy没有明确的Character。但是可以强行声明。

```groovy
char c1 = 'A' 
assert c1 instanceof Character

def c2 = 'B' as char 
assert c2 instanceof Character

def c3 = (char)'C' 
assert c3 instanceof Character
```



4.List

Groovy的列表和python的很像。支持动态扩展，支持放置多种数据。使用方法支持def和直接定义。还可以像python那样索引

```groovy
//List中存储任意类型
def heterogeneous = [1, "a", true]

//判断List默认类型
def arrayList = [1, 2, 3]
assert arrayList instanceof java.util.ArrayList

//使用as强转类型
def linkedList = [2, 3, 4] as LinkedList    
assert linkedList instanceof java.util.LinkedList

//定义指定类型List
LinkedList otherLinked = [3, 4, 5]          
assert otherLinked instanceof java.util.LinkedList

// 像python一样索引
assert letters[1] == 'b'
//负数下标则从右向左index
assert letters[-1] == 'd'    
assert letters[-2] == 'c'
//指定item赋值判断
letters[2] = 'C'             
assert letters[2] == 'C'
//给List追加item
letters << 'e'               
assert letters[ 4] == 'e'
assert letters[-1] == 'e'
//获取一段List子集
assert letters[1, 3] == ['b', 'd']         
assert letters[2..4] == ['C', 'd', 'e'] 
```



5.Map

```groovy
//定义一个Map
def colors = [red: '#FF0000', green: '#00FF00', blue: '#0000FF']   
//获取一些指定key的value进行判断操作
assert colors['red'] == '#FF0000'    
assert colors.green  == '#00FF00'
```


6.运算符

- **： 次方运算符。
- ?.：安全占位符。和kotlin一样避免空指针异常。
- .@：直接域访问操作符。因为Groovy自动支持属性getter方法，但有时候我们有一个自己写的特殊getter方法，当不想调用这个特殊的getter方法则可以用直接域访问操作符。这点跟kotlin的
- .&：方法指针操作符，因为闭包可以被作为一个方法的参数，如果想让一个方法作为另一个方法的参数则可以将一个方法当成一个闭包作为另一个方法的参数。
- ?:：二目运算符。与kotlin中的类似。
- `*.`展开运算符，一个集合使用展开运算符可以得到一个元素为原集合各个元素执行后面指定方法所得值的集合。

```groovy
cars = [
    new Car(make: 'Peugeot', model: '508'),
   null,                                              
   new Car(make: 'Renault', model: 'Clio')]
assert cars*.make == ['Peugeot', null, 'Renault']     
assert null*.make == null 
```

7.闭包
groovy里比较重要的是闭包的概念。官方定义是“Groovy中的闭包是一个开放，匿名的代码块，可以接受参数，返回值并分配给变量”。
其实闭包跟kotlin的lambda函数很像，都是先定义后执行。但是又有一些细微的区别。接下来我们细讲讲gradle的闭包。

闭包是可以用作方法参数的代码块，Groovy的闭包更象是一个代码块或者方法指针，代码在某处被定义然后在其后的调用处执行。一个闭包实际上就是一个Closure类型的实例。写法和kotlin的lambda函数很像。

我们常见的闭包是这样的
```groovy
//最基本的闭包
{ item++ }                                          
//使用->将参数与代码分离
{item -> item++ }                                       
//使用隐含参数it
{ println it }                               
//使用显示的名为参数
{ name -> println name }

// 调用方法
a.call()
a()

// Groovy的闭包支持最后一个参数为不定长可变长度的参数。
def multiConcat = { int n, String... args ->                
    args.join('')*n
}
```
大家要注意，如果我们单纯的只是写成 a = { item++ }, 这只是定义了一个闭包，是不能运行的。必须调用a.call()才能运行出来。所以大家可以理解了，闭包就是一段代码块而已。当我们有需要的时候，可以去运行它，这么一想是不是和lambda函数很像？

如果你看了[官网](https://docs.gradle.org/current/dsl/index.html)，你会发现有一些这样的说法，

![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/20190830093028.png)

什么叫做delegate？这里涉及到闭包内部的三种对象。
- this 对应于定义闭包的那个类，如果在内部类中定义，指向的是内部类
- owenr 对应于定义闭包的那个类或者闭包，如果在闭包中定义，对应闭包，否则同this一致
- delegate 默认是和owner一致，或者自定义delegate指向

this和owner都比较好理解。我们可以用闭包的getxxx方法获取
```groovy
def thisObject = closure.getThisObject()
def ownerObject = closure.getOwner()
def delegate = closure.getDelegate()
```

重头戏还是delegate这个对象。闭包可以设置delegate对象，**设置delegate的意义就是将闭包和一个具体的对象关联起来**。
我们先来看个例子，这里以自定义android闭包为例。
```
android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"

    defaultConfig {
        minSdkVersion 15
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
    }
}
```

这个闭包对应的实体类是两个。
```
# Android.groovy
class Android {
    private int mCompileSdkVersion
    private String mBuildToolsVersion
    private ProductFlavor mProductFlavor

    Android() {
        this.mProductFlavor = new ProductFlavor()
    }

    void compileSdkVersion(int compileSdkVersion) {
        this.mCompileSdkVersion = compileSdkVersion
    }

    void buildToolsVersion(String buildToolsVersion) {
        this.mBuildToolsVersion = buildToolsVersion
    }

    void defaultConfig(Closure closure) {
        closure.setDelegate(mProductFlavor)
        closure.setResolveStrategy(Closure.DELEGATE_FIRST)
        closure.call()
    }
    
    @Override
     String toString() {
        return "Android{" +
                "mCompileSdkVersion=" + mCompileSdkVersion +
                ", mBuildToolsVersion='" + mBuildToolsVersion + '\'' +
                ", mProductFlavor=" + mProductFlavor +
                '}'
    }
}

# ProductFlavor.groovy
class ProductFlavor {
    private int mVersionCode
    private String mVersionName
    private int mMinSdkVersion
    private int mTargetSdkVersion

    def versionCode(int versionCode) {
        mVersionCode = versionCode
    }

    def versionName(String versionName) {
        mVersionName = versionName
    }

    def minSdkVersion(int minSdkVersion) {
        mMinSdkVersion = minSdkVersion
    }


    def targetSdkVersion(int targetSdkVersion) {
        mTargetSdkVersion = targetSdkVersion
    }

    @Override
    String toString() {
        return "ProductFlavor{" +
                "mVersionCode=" + mVersionCode +
                ", mVersionName='" + mVersionName + '\'' +
                ", mMinSdkVersion=" + mMinSdkVersion +
                ", mTargetSdkVersion=" + mTargetSdkVersion +
                '}'
    }
}
```

然后定义的时候就写成
```
//闭包定义
def android = {
        compileSdkVersion 25
        buildToolsVersion "25.0.2"
        defaultConfig {
            minSdkVersion 15
            targetSdkVersion 25
            versionCode 1
            versionName "1.0"
        }
    }

//调用
Android bean = new Android()
android.delegate = bean
android.call()
println bean.toString()

//打印结果
Android{mCompileSdkVersion=25, mBuildToolsVersion='25.0.2', mProductFlavor=ProductFlavor{mVersionCode=1, mVersionName='1.0', mMinSdkVersion=15, mTargetSdkVersion=25}}
```
这样就能将闭包中声明的值，赋给两个对象Android和ProductFlavor来处理了。


上面官网的图里，说ScriptHandler被设置成buildscript的delegate。意思就是buildscript定义的参数被ScriptHandler拿来使用了。大家有兴趣的可以去看看ScriptHandler的源码~


## Project与Task-gradle构建体系
上面我们讲完了基本的用法，大家可能懂gradle的配置和写法了。但是可能还是不懂gradle的构建体系到底是怎么样的。这里我们就要深入进gradle的构建体系Project和Task了。下面的东西看着就要动动脑筋了。

1.Task
Task是gradle脚本中的最小可执行单元。类图如下：
![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/006tNc79ly1g2ynzenpu3j30ji0cijsn.jpg)

值得注意的是因为Gradle构建脚本默认的名字是build.gradle，当在shell中执行gradle命令时，Gradle会去当前目录下寻找名字是build.gradle的文件。**所以只有定义在build.gradle中的Task才是有效的。**

可以通过三种方式来声明task。我们可以根据自己的项目需要去定义Task。比如自定义task接管gradle的编译过程
```groovy
task myTask2 << {
    println "doLast in task2"
}

//采用 Project.task(String name) 方法来创建
project.task("myTask3").doLast {
    println "doLast in task3"
}

//采用 TaskContainer.create(String name) 方法来创建
project.tasks.create("myTask4").doLast {
    println "doLast in task4"
}
```

TaskContianer 是用来管理所有的 Task 实例集合的，可以通过 Project.getTasks() 来获取 TaskContainer 实例。
常见接口：
```java
findByPath(path: String): Task
getByPath(path: String): Task
getByName(name: String): Task
withType(type: Class): TaskCollection
matching(condition: Closure): TaskCollection

//创建task
create(name: String): Task
create(name: String, configure: Closure): Task 
create(name: String, type: Class): Task
create(options: Map<String, ?>): Task
create(options: Map<String, ?>, configure: Closure): Task

//当task被加入到TaskContainer时的监听
whenTaskAdded(action: Closure)
```

Gradle支持增量编译。了解过编译profile文件的朋友都知道，里面有大量的task都是`up-to-date`。那么这种up-to-date是什么意思呢。`Gradle的Task会把每次运行的结果缓存下来，当下次运行时，会检查一个task的输入输出有没有变更。如果没有变更就是up-to-date，跳过编译。`

![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/006tNc79ly1g2zg1osecqj30rs0dtwfo.jpg)


2.Project
先从Project对象讲起，Project是与Gradle交互的主接口。android开发中最为我们所熟悉的就是build.gradle文件，这个文件与Project是一对一的关系，build.gradle文件是project对象的委托，脚本中的配置都是对应着Project的Api。Gradle构建进程启动的时候会根据build.gradle去实例化Project类。也就是说，构建的时候，每个build.gradle文件会生成一个Project对象，这个对象负责当前module的构建。

Project本质上是包含多个Task的容器，所有的Task存在TaskContainer中。我们从名字可以看出
![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/006tNc79ly1g2ynlq52cbj30gb0xxjv3.jpg)

可以看到dependencies, configuration, allprojects, subprojects, beforeEvaluate, afterEvaluate这些都是我们常见的配置项，在build.gradle文件中接收一个闭包Closure。

好了，现在我们已经聊了build.gradle了，但是大家都知道，我们项目中还有一个settings.gradle呢，这个是拿来干嘛的呢？这就要说到Project的`Lifecycle`了，也就是Gradle构建Project的步骤，看官网原文：

- Create a Settings instance for the build.
- Evaluate the settings.gradle script, if present, against the Settings object to configure it.
- Use the configured Settings object to create the hierarchy of Project instances.
- Finally, evaluate each Project by executing its build.gradle file, if present, against the project. The projects are evaluated in breadth-wise order(宽度搜索), such that a project is evaluated before its child projects. This order can be overridden by calling `Project.evaluationDependsOnChildren()` or by adding an explicit evaluation dependency using `Project.evaluationDependsOn(java.lang.String)`.

也就是说，Project对象依赖Settings对象的构建。我们常在settings.gradle文件中配置需要引入的module，就是这个原因。


3.Property
看完了build.gradle和settings.gradle，接下来我们讲讲gradle.properties。这个文件存放的键值对形式的属性，这些属性能被项目中的gradle脚本使用ext.xxx所访问。

我们也可以使用Properties类来动态创建属性文件。如：
```groovy
def defaultProps = new Properties()
defaultProps.setProperty("debuggable", 'true')
defaultProps.setProperty("groupId", GROUP)
```

并且属性可以继承，在一个项目中定义的属性可以自动被子项目继承。所以在哪个子项目都可以使用project.ext.xxx访问。不同子项目间采用通用的配置插件来配置
```groovy
apply from: rootProject.file('library.gradle')
```

## 总结
通过上面的学习，大家应该已经了解了gradle的基本配置，写法和比较浅显的内部原理了。因为篇幅原因，深入的内容我们放在下一篇。敬请期待《一篇文章深入gradle》

我是Android笨鸟之旅，一个陪着你慢慢变强的公众号。


参考：
[官网](https://docs.gradle.org/current/dsl/index.html)
[[Android Gradle] 搞定Groovy闭包这一篇就够了](https://www.jianshu.com/p/6dc2074480b8)
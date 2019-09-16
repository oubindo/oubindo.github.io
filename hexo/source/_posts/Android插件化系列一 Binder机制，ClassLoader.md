---
title: Android插件化系列一：开篇前言，Binder机制，ClassLoader
date: 2019-09-07 18:20:01
tags: 插件化
---

# Android插件化系列一: 开篇前言，Binder机制，ClassLoader

## 系列前言

Hello，美好的一周又开始啦，让我们开始愉快的学习吧。

从今天开始，我会花较多的时间来跟大家一起学习Android插件化。这一篇文章是Android插件化的启动篇。

Android插件化是之前几年里的一个很火的技术概念。从2012年开始就有人在研究这门技术。从粗糙的AndroidDynamicLoader框架，到第一代的DroidPlugin等，继而发展到第二代的VirtualApk，Replugin等，再到现如今的VirtualApp，Atlas。插件化在国内逐渐的发展和完善，却也在近几年出现了RN等替代品以后慢慢会走向弱势。

尽管插件化技术的研究热潮已经过去，但是这门技术本身还是有着大量的技术实践，对于我们了解Android机制很有帮助。所以从这篇文章开始我会写一系列的文章，加上自己对插件化的实践，最后会去从源码角度分析几个优秀的插件化库，形成一套完整的插件化的理论体系。

下面是插件化的技术框架，也是我这个系列文章的行文思路，

![Android插件化文章框架](https://ws1.sinaimg.cn/large/006tKfTcly1g0qgbl8170j31390u0gu5.jpg)

## 一. Binder机制

网上分析Binder机制的文章已经很多了，在这篇文章里，我不会去讲解Binder的使用，而是会去讲解清楚Binder的设计思路，设计原理和对于插件化的使用。

### 为什么需要Binder

首先我们知道，Android是基于Linux内核开发的。对于Linux来说，操作系统为一个二进制可执行文件创建了一个载有该文件自己的栈，堆、数据映射以及共享库的内存片段，还为其分配特殊的内部管理结构。这就是一个进程。操作系统必须提供公平的使用机制，使得每个进程能正常的开始，执行和终结。

这样呢，就引入了一个问题。一个进程能不能去操作别的进程的数据呢？我们可以想一下，这是绝对不能出现的，尤其是系统级的进程，如果被别的进程影响了可能会造成整个系统的崩塌。所以我们很自然的想到，我们应该把进程隔离起来，linux也是这样做的，它的虚拟内存机制为每个进程分配连续的内存空间，进程只能操作自己的虚拟内存空间。同时，还必须满足进程之间保持通信的能力，毕竟团结力量大，单凭单个进程的独立运作是不能撑起操作系统的功能需求的。

为了解决这个问题，Linux引进了用户空间User Space和内核空间Kernel Space的区别。用户空间要想访问内核空间，唯一方式就是系统调用。内核空间通过接口把应用程序请求传给内核处理后返回给应用程序。同时，用户空间进程如果想升级为内核空间进程，需要进行安全检查。

> 补充知识：系统调用主要通过两个方法：
>
> - copy_from_user（）：将用户空间的数据拷贝到内核空间
>
> - copy_to_user（）：将内核空间的数据拷贝到用户空间

![linux系统调用](https://ws4.sinaimg.cn/large/006tKfTcly1g0qhlh17nfj30bn0c4glx.jpg)

以上就是linux系统的跨进程通信机制。而我们马上要说的Binder，就是跨进程的一种方式

### Binder模型

Binder是一种进程间通信(IPC)方式，Android常见的进程中通信方式有文件共享，Bundle，AIDL，Messenger，ContentProvider，Socket。其中AIDL，Messenger，ContentProvider都是基于Binder。Linux系统通过Binder驱动来管理Binder机制。

> Binder的实现基于mmap()系统调用，只用拷贝一次，比常规文件页操作更快。微信开源的MMKV等也是基于此。有兴趣的可以了解一下。

首先Binder机制有四个参与者，Server，Client两个进程，ServiceManager，Binder驱动(内核空间)。其中ServiceManager和Binder驱动都是系统实现的，而Server和Client是需要开发者自己实现的。四者之中只有Binder驱动是运行在内核空间的。

![Binder机制](https://ws2.sinaimg.cn/large/006tKfTcly1g0r3hyi4obj30k00br74u.jpg)

这里的ServiceManager作为Manager，承担着Binder通信的建立，Binder的注册和传递的能力。Service负责创建Binder，并为他起一个字符形式的名字，然后把Binder和名字通过**通过Binder驱动，借助于ServiceManager自带的Binder**向ServiceManager注册。注意这里，因为Service和ServiceManager也是跨进程通信需要Binder，ServerManager是自带Binder的，所以相对ServiceManager来说Service也就相当于Client了。

Service注册了这个Binder以后，Client就能通过名字获得Binder的引用了。这里的跨进程通信双方就变成了Client和ServiceManager，然后ServiceManager从Binder表取出**Binder的引用**返给Client，这样的话如果有多个Client的话，多次返回引用就行了，但是事实上引用的都是放在ServiceManager中的Service。

当Client经过Binder驱动跟Service通信的时候，往往需要获取到Service的某个对象object。这时候为了安全考虑，Binder会把object的代理对象proxyobject返回，这个对象拥有一模一样的方法，但是没有具体能力，只负责接收参数传给真正的object使用。

所以完整的Binder通信过程是

![binder模型](https://ws4.sinaimg.cn/large/006tKfTcly1g0rnpftgxij30dn0cjaae.jpg)

OK，跨进程通信就讲清楚了。接下来我们讲讲插件化中的Binder。

### Binder与插件化

首先，我们先回顾一下Activity的启动过程，Instrumentation调用了ActivityManagerNative，这个AMN是我们的本地对象，然后AMN调用getDefault拿到了ActivityManagerProxy，这个人AMP就是AMS在本地的代理。相当于binder模型中的Client，而这个AMP继承的是IActivityManager，拥有四大组件的所有需要AMS参与的方法。本地通过调用这个AMP方法来间接地调用AMS。这样，我们就调用到了AMS启动了Activity。

那么，AMS如何与Client进行通信呢？现在我们通过Launcher启动了Activity，肯定要告诉Launcher “没你什么事了，你洗洗睡吧”。大家可以看到，这里双方的角色就发生了改变，AMS需要去发消息，承担Client的角色，而Launcher这时候作为Service提供服务。而这次通信同样也是使用的Binder机制。AMS这边保存了一个ApplicationThreadProxy对象，这个对象就是Launcher的ApplicationThread的代理。AMS通过ATP给App发消息，App通过ApplicationThread处理。

![Activity启动流程](https://ws1.sinaimg.cn/large/006tKfTcly1g0wlwfnzgtj31n30u0gy7.jpg)

以上，就是Binder机制在Android中的运用，我们后面会通过hook这个过程实现插件化。

### 总结

看了这么多，可能还是很多朋友不懂Binder。我也是这样，很长一段时间都不知道Binder到底指的是啥。后来我看到了这样一种定义：

- 从进程间通信的角度看，Binder 是一种进程间通信的机制；

- 从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象；

- 从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理

- 从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会对这个跨越进程边界的对象对一点点特殊处理，自动完成代理对象和本地对象之间的转换。



## 二. ClassLoader

### 双亲委托模型

Java中默认有三种ClassLoader。分别是:

- BootStrap ClassLoader：启动类加载器，最顶层的加载器。主要负责加载JDK中的核心类。在JVM启动后也随着启动，并构造Ext ClassLoader和App ClassLoader。
- Extension ClassLoader：扩展类加载器，负责加载Java的扩展类库。
- App ClassLoader：系统类加载器，负责加载应用程序的所有jar和class文件。
- 自定义ClassLoader：需要继承自ClassLoader类。

ClassLoader默认使用双亲委托模型来搜索类。每个ClassLoader都有一个父类的引用。当ClassLoader需要加载某个类时，先判断是否加载过，如果加载过就返回Class对象。否则交给他的父类去加载，继续判断是否加载过。这样 `层层判断`，就到了最顶层的BootStrap ClassLoader来试图加载。如果连最顶层的Bootstrap ClassLoader都没加载过，那就加载。如果加载失败，就转交给子ClassLoader，`层层加载`，直到最底层。如果还不能加载的话那就只能抛出异常了。

通过这种双亲委托模型，好处是：

- 更高效，父类加载一次就可以避免了子类多次重复加载
- 更安全，避免了外界伪造java核心类。



### Android中的ClassLoader

android从5.0开始使用art虚拟机，这种虚拟机在程序运行时也需要ClassLoader将类加载到内存中，但是与java不同的是，java虚拟机通过读取class字节码来加载，但是art则是通过dex字节码来加载。这是一种优化，可以合并多个class文件为一个classes.dex文件。

android一共有三种类加载器:

- BootClassLoader：父类构造器

- PathClassLoader：一般是加载指定路径/data/app中的apk，也就是安装到手机中的apk。所以一般作为默认的加载器。

- DexClassLoader：从包含classes.dex的jar或者apk中，加载类的加载器，可用于动态加载。

  

看PathClassLoader和DexClassLoader源码，都是继承自BaseDexClassLoader。

```java
public DexClassLoader(String dexPath, String optimizedDirectory,
      String librarySearchPath, ClassLoader parent) {
    super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
}


public class PathClassLoader extends BaseDexClassLoader {

    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);  //见下文
    }
}

public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String libraryPath, ClassLoader parent) {
        super(parent);  //见下文
        //收集dex文件和Native动态库【见小节3.2】
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
}
```

我们可以看到PathClassLoader的两个参数都为null，表明只能接受固定的dex文件，而这个文件是只能在安装后出现的。而DexClassLoader中optimizedDirectory，和librarySearchPath都是可以自己定义的，说明我们可以传入一个jar或者apk包，保证解压缩后是一个dex文件就可以操作了。因此，我们通常使用DexClassLoader来进行插件化和热修复。

可以看到，BaseDexClassLoader有一个相当重要的过程就是初始化DexPathList。初始化DexPathList的过程主要是收集dexElements和nativeLibraryPathElements。一个Classloader可以包含多个dex文件，每个dex文件被封装到一个Element对象。这element对象在初始化和热修复逻辑中是相当重要的。当查找某个类时，会遍历dexElements，如果找到就返回，否则继续遍历。所以当多个dex中有相同的类，只会加载前面的dex中的类。下面是这段逻辑的具体实现

```java
public Class findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            //找到目标类，则直接返回
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    return null;
}
```



### 总结

我们先是讲解了Java中类加载的双亲委托机制，然后介绍了Android中的几种ClassLoader，从源码角度介绍了两种ClassLoader加载机制的不同。以后的插件化实践中，我们会经常用到DexClassLoader。



### 参考资料

[Android跨进程通信：图文详解 Binder机制 原理](https://blog.csdn.net/carson_ho/article/details/73560642)

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

[深入理解Android中的ClassLoader](https://juejin.im/post/5a28e7e86fb9a045117105c3)

[Android类加载器ClassLoader](http://gityuan.com/2017/03/19/android-classloader/)
---
title: 一篇文章深入gradle（上篇）:依赖机制 
date: 2019-09-05 21:30:44
tags: gradle
---

## 一篇文章深入gradle（上篇）:依赖机制 
Hello，各位朋友们，小笨鸟又和你们见面啦。不同于网上泛泛而谈的入门文章只停留在“怎么用”的层次，本篇文章从源码角度去理解gradle。当然，如果你没有看过我的前一篇文章《一篇文章基本看懂gradle》，还是建议你先看一下，本篇文章的很多知识点会在上篇文章的基础上展开。

本篇文章会比较深入，需要费一些脑筋和精力才能理解。建议跟着我的行文思路一步一步理解。本文的篇幅会比较长。阅读大概需要花费半个小时。阅读本文将会了解:

- gradle构建Lifecycle
- gradle Extension和Plugin
- gradle依赖实现，Configuration

其中gradle依赖将会是本文的重点。本来之前还想把artifact也讲解到，但是发现篇幅已经很长了，所以就一篇文章拆成两篇吧。

## gradle构建Lifecycle
Lifecycle的概念我们在上一篇文章讲Project的时候也讲到过。在这篇文章中，我们会再简单讲一讲作为引入。

正如[官网](https://docs.gradle.org/current/userguide/build_lifecycle.html)所说，gradle构建流程总共为三个阶段：
- 初始化阶段：在这个阶段中settings.gradle是主角，gradle会把settings.gradle中的配置代理给Settings类。主要负责的是判断哪些module需要参与到构建中，然后根据这些module的设置初始化他们的delegate对象Project。注意，Project对象是在初始化阶段生成的。
- 配置阶段：配置阶段的任务是执行各项目下的build.gradle，完成Project对象的配置，并且构造Task任务依赖关系图`TaskExectionGraph`以便在执行阶段按照依赖关系执行Task。
- 执行阶段：执行阶段会根据你命令行输入的Task，按照依赖关系通过调用`gradle taskname`进行执行。

值得一提的是，我们可以通过多种方式对project构建阶段进行监控和处理。如
```java
// 以下为Project的方法
afterEvaluate(closure)，afterEvaluate(action)
beforeEvaluate(closure)，beforeEvaluate(action)

// 以下为gradle提供的生命周期回调
afterProject(closure)，afterProject(action)
beforeProject(closure)，beforeProject(action)
buildFinished(closure)，buildFinished(action)
projectsEvaluated(closure)，projectsEvaluated(action)
projectsLoaded(closure)，projectsLoaded(action)
settingsEvaluated(closure)，settingsEvaluated(action)
addBuildListener(buildListener)
addListener(listener)
addProjectEvaluationListener(listener)

// Task也有这种方法
afterTask​(Closure closure)
beforeTask(Closure closure)
//任务准备好后调用
whenReady(Closure closure)
```

那么知道这样的构建流程我们可以怎么使用呢？我们可以进行监控，或者是动态的根据需要去控制project和task的构建执行。比如为了加快编译速度，我们去掉一些测试的task。就可以这样写
```
gradle.taskGraph.whenReady {
        tasks.each { task ->
            if (task.name.contains("Test")) {
                task.enabled = false
            } else if (task.name == "mockableAndroidJar") {
                task.enabled = false
            }
        }
    }
```


## Extension和Plugin
Extension和Plugin都是我们日常开发中经常有讲到的东西。上一篇文章中我们讲到了build.gradle中的闭包都是有一个Delegate代理的，这个代理对象可以接受闭包中的参数传递给自己来使用，那么这个代理是啥呢？其实就是我们这里要说的Extension。

1.Extension通俗来讲其实就是一个普通的java bean。里面存放一些参数。使用需要借助于ExtensionContainer来进行创建。举个栗子
```java
// 先定义一个实体类
public class Message {
    String message = "empty"
    String greeter = "none"
}

// 接下来在gradle文件中去使用project.extensions(也就是ExtensionContainer)来进行创建。
def extension = project.extensions.create("cyMessage", Message)

// 再写个task来进行验证
project.task('sendMessage') {
    doLast {
        println "${extension.message} from ${extension.greeter}"
        println project.cyMessage.message + "from ${extension.greeter}"
    }
}
```

2.Plugin插件
讲完了Extension，我们可能就会疑惑了。因为我们在build.gradle中并没有看到这些javabean和extension添加的操作啊，那么这些代码是在哪里写的呢？这时候我们就要引入Plugin的概念了，也就是你看到的
```
apply plugin 'com.android.application'
```
这个玩意了。这个就是Android Application的插件。android相关的Extension都是定义在这个插件中的。
有些同学可能对插件不是很理解，为什么需要这个东西。其实大家可以思考一下，gradle只是一个通用的构建工具。在他上层可能有各种应用，比如java，比如Android，甚至可能是未来的鸿蒙。那么这些应用对gradle肯定会有不同的扩展，又肯定不能把这些扩展直接放在gradle中。所以在上层添加插件就是个好选择，需要什么扩展就选什么插件。
Android开发中常见的插件有三种：
```
apply plugin 'com.android.application'   // AppPlugin,  主module才能引用
apply plugin 'com.android.library'    // LibraryPlugin, 普通的android module引用
apply plugin 'java'                           // java module的引用
```

插件的其它知识不是本篇文章的重点，网上这方面的文章很多，大家可以自行学习。我们根据现在掌握的知识就要去探索gradle更深层次的知识啦。

## gradle依赖实现
这两篇文章截止到现在，gradle构建相关的流程我们基本都走通了。但是有个很重要的组件我们没有去分析。就是依赖管理。我们前一篇文章也讲到了，依赖管理是gradle的一个很重要的特性，方便我们进行代码复用。那么我们这一部分就专门讲一讲依赖管理到底是怎么实现的。

首先我们还是看看基本的代码
```
dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    implementation 'com.android.support:appcompat-v7:26.1.0'
    api  'com.android.support:appcompat-v7:26.1.0'
    implementation project(':test')
    androidTestImplementation 'com.android.support.test:runner:1.0.1'
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
}
```
我们可以看到，一般有三种三方库类型，第一种是二进制文件，fileTree指向这个文件夹下的所有jar文件，第二种是三方库的string坐标形式，第三种是project的形式。
当然，我们也会知道，代码中的implementation和api两种引用方式是有区别的。implement的依赖是不能传递的，但是api是可以的。

我们在之前的研发中，可能没有去考虑深层次的这三种三方库类型，两种引用方式的原因。当我们按照上一篇中所说的代理模式去DependencyHandler中找implement, api等相关方法的时候，却发现，好像并没有定义这些方法，那么这是怎么回事呢？如果没有定义，是不是可以随便定义一种引用方式呢？带着问题我们往下面看

### 1.源码阅读姿势
首先我们还是先介绍一下阅读gradle源码的方式。我尝试了很多种方式，发现还是Android studio阅读起来最舒服，然后找到了一种方法。就是可以建一个gradle的demo，然后建一个module，把除了build.gradle以外的东西全部删掉。然后拷贝下面的代码进去。这样就能看到源码了
```
apply plugin 'java'

dependencies {
     compile gradleApi()
     compile 'xxxx'  // 填入你gradle的版本
}
```

### 2.源码分析
#### 2.1 methodmissing  
首先我们先介绍一个groovy语言的特性：methodmissing。大家可以参考[官网](http://groovy-lang.org/metaprogramming.html#_methodmissing)
简单来说就是当我们预先在一个类中定义一个methodmissing方法。然后在这个类的对象上调用之前没有定义过的方法时，这个方法就会降级(fallback)到它所定义的methodmissing方法上。
```
class GORM {

   def dynamicMethods = [...] // an array of dynamic methods that use regex

   def methodMissing(String name, args) {
       def method = dynamicMethods.find { it.match(name) }
       if(method) {
          GORM.metaClass."$name" = { Object[] varArgs ->
             method.invoke(delegate, name, varArgs)
          }
          return method.invoke(delegate,name, args)
       }
       else throw new MissingMethodException(name, delegate, args)
   }
}

assert new GORM().methodA(1) == resultA
```
如图，当我们调用methodA时，因为这个方法没有定义，就会转到methodmissing方法上，并且会把这个方法的名字methodA和它的参数一起传到methodmissing，这样如果dynamicMethod里面有定义methodA的话，这个方法就能执行了。这就是methodmissing的妙用。

为什么需要这种机制呢？我理解这还是为了扩展性。dependencies是gradle自身的功能。它不能完全的总括所有上层应用可能会有的引用方式。每种插件都可能增加引用的方式，为了扩展性考虑，必须采用这种methodmissing的特性，把这些引用交给插件处理。比如Android的implement, api, annotationProcessor等。

#### 2.2 Configuration  
在往下面讲解前，我们先了解一下[Configuration](https://docs.gradle.org/current/userguide/managing_dependency_configurations.html)的一些知识。  
按照官网所说：
> Every dependency declared for a Gradle project applies to a specific scope. For example some dependencies should be used for compiling source code whereas others only need to be available at runtime. Gradle represents the scope of a dependency with the help of a Configuration. Every configuration can be identified by a unique name.

也就是说，Configuration定义了依赖在编译和运行时候的不同范围, 每个Configuration都有name来区分。比如android常见的两种依赖方式implementation和api。
![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/20190904002104.png)。


#### 2.3 依赖的识别
gradle中使用MethodMixIn这个接口来实现methodmissing的能力。
```java
// MethodMixIn
public interface MethodMixIn {
    MethodAccess getAdditionalMethods();
}
```

```java
public interface MethodAccess {
    /**
     * Returns true when this object is known to have a method with the given name that accepts the given arguments.
     *
     * <p>Note that not every method is known. Some methods may require an attempt invoke it in order for them to be discovered.</p>
     */
    boolean hasMethod(String name, Object... arguments);

    /**
     * Invokes the method with the given name and arguments.
     */
    DynamicInvokeResult tryInvokeMethod(String name, Object... arguments);
}
```


可以看到这里的methodmissing主要是在找不到这个方法的时候去返回一个MethodAccess，MethodAccess中去判断是否存在以及动态执行这个method。

接下来我们看DependencyHandler的实现类DefaultDependencyHandler。这个类实现了MethodMixIn接口，返回的是一个DynamicAddDependencyMethods对象。
```java
// DefaultDependencyHandler.java
public DefaultDependencyHandler(...) {
    ...
    this.dynamicMethods = new DynamicAddDependencyMethods(configurationContainer, new DefaultDependencyHandler.DirectDependencyAdder());
}

public MethodAccess getAdditionalMethods() {
    return this.dynamicMethods;
}
```
所以其实就是返回了一个DynamicAddDependencyMethods去加以判断。那么毫无疑问要在这个类中进行判断和执行具体方法。接下来我们看看这个类中是怎么处理的。
```java
    DynamicAddDependencyMethods(ConfigurationContainer configurationContainer, DynamicAddDependencyMethods.DependencyAdder dependencyAdder) {
        this.configurationContainer = configurationContainer;
        this.dependencyAdder = dependencyAdder;
    }

    public boolean hasMethod(String name, Object... arguments) {
        return arguments.length != 0 && this.configurationContainer.findByName(name) != null;
    }

    public DynamicInvokeResult tryInvokeMethod(String name, Object... arguments) {
        if (arguments.length == 0) {
            return DynamicInvokeResult.notFound();
        } else {
            Configuration configuration = (Configuration)this.configurationContainer.findByName(name);
            if (configuration == null) {
                return DynamicInvokeResult.notFound();
            } else {
                List<?> normalizedArgs = CollectionUtils.flattenCollections(arguments);
                if (normalizedArgs.size() == 2 && normalizedArgs.get(1) instanceof Closure) {
                    return DynamicInvokeResult.found(this.dependencyAdder.add(configuration, normalizedArgs.get(0), (Closure)normalizedArgs.get(1)));
                } else if (normalizedArgs.size() == 1) {
                    return DynamicInvokeResult.found(this.dependencyAdder.add(configuration, normalizedArgs.get(0), (Closure)null));
                } else {
                    Iterator var5 = normalizedArgs.iterator();

                    while(var5.hasNext()) {
                        Object arg = var5.next();
                        this.dependencyAdder.add(configuration, arg, (Closure)null);
                    }

                    return DynamicInvokeResult.found();
                }
            }
        }
    }
```


可以看到这个类的两个要点：  
1.判断Configuration有无：通过外部传入的ConfigurationContainer来判断是否存在这个方法。**这样我们可以联想到，这个ConfigurationContainer肯定是每个平台Plugin自己传入的，必须是已定义的才能使用**。比如android就添加了implementation, api等。如果你想查看Configuration在gradle源码中的初始化和配置，可以查看VariantDependencies这个类。

2.执行方法：真正的执行方法会根据参数来判断，比如我们常见的一个参数的引用形式，还有一个参数+一个闭包的形式，比如
```java
compile('com.zhyea:ar4j:1.0') {
		exclude module: 'cglib' //by artifact name
}
```
这种类型的引用。当在ConfigurationContainer中找到了这个引用方式(以下都称Configuration)时，就会返回一个DynamicInvokeResult。具体这个类的作用我们后面再看，我们先看他们都做了一个
```java
this.dependencyAdder.add(configuration, arg, (Closure)null);
```
的操作，这个操作是做了些什么呢，我们继续往下跟就会发现，其实还是调用了DefaultDependencyHandler的doAdd方法。
```java
    // DefaultDependencyHandler.java
    private class DirectDependencyAdder implements DependencyAdder<Dependency> {
        private DirectDependencyAdder() {
        }

        public Dependency add(Configuration configuration, Object dependencyNotation, @Nullable Closure configureAction) {
            return DefaultDependencyHandler.this.doAdd(configuration, dependencyNotation, configureAction);
        }
    }

    private Dependency doAdd(Configuration configuration, Object dependencyNotation, Closure configureClosure) {
        if (dependencyNotation instanceof Configuration) {
            Configuration other = (Configuration)dependencyNotation;
            if (!this.configurationContainer.contains(other)) {
                throw new UnsupportedOperationException("Currently you can only declare dependencies on configurations from the same project.");
            } else {
                configuration.extendsFrom(new Configuration[]{other});
                return null;
            }
        } else {
            Dependency dependency = this.create(dependencyNotation, configureClosure);
            configuration.getDependencies().add(dependency);
            return dependency;
        }
    }
```
可以看到，这里会先判断dependencyNotation是否是Configuration，如果存在的话，就让当前的configuration继承dependencyNotation，也就是所有添加到dependencyNotation的依赖都会添加到configuration中。

这里可能有些朋友就会疑惑了，为啥还要对dependencyNotation判断呢？这个主要是为了处理嵌套的情况。比如implementation project(path: ':projectA', configuration: 'configA')这种类型的引用。有兴趣可以看看上面的`CollectionUtils.flattenCollections(arguments)`方法。

总结一下，这个过程就是借助gradle的MethodMixIn接口，将所有未定义的引用方法转到getAdditionalMethods方法上来，在这个方法里面判断Configuration是否存在，如果存在的话就生成Dependency。

#### 2.4 依赖的创建
可以看到上面过程的最后，是DefaultDependencyHandler调用了create方法创建出了一个Dependency。我们继续来分析创建Dependency的过程。

```java
    // DefaultDependencyHandler.java
    public Dependency create(Object dependencyNotation, Closure configureClosure) {
        Dependency dependency = this.dependencyFactory.createDependency(dependencyNotation);
        return (Dependency)ConfigureUtil.configure(configureClosure, dependency);
    }

    // DefaultDependencyFactory.java
    public Dependency createDependency(Object dependencyNotation) {
        Dependency dependency = (Dependency)this.dependencyNotationParser.parseNotation(dependencyNotation);
        this.injectServices(dependency);
        return dependency;
    }
```
可以看到最终是调用了dependencyNotationParser来parse这个dependencyNotation。而这里的dependencyNotationParser其实就是DependencyNotationParser这个类。

```java
public class DependencyNotationParser {
    public static NotationParser<Object, Dependency> parser(Instantiator instantiator, DefaultProjectDependencyFactory dependencyFactory, ClassPathRegistry classPathRegistry, FileLookup fileLookup, RuntimeShadedJarFactory runtimeShadedJarFactory, CurrentGradleInstallation currentGradleInstallation, Interner<String> stringInterner) {
        return NotationParserBuilder.toType(Dependency.class)
        .fromCharSequence(new DependencyStringNotationConverter(instantiator, DefaultExternalModuleDependency.class, stringInterner))
        .converter(new DependencyMapNotationConverter(instantiator, DefaultExternalModuleDependency.class))
        .fromType(FileCollection.class, new DependencyFilesNotationConverter(instantiator))
        .fromType(Project.class, new DependencyProjectNotationConverter(dependencyFactory))
        .fromType(ClassPathNotation.class, new DependencyClassPathNotationConverter(instantiator, classPathRegistry, fileLookup.getFileResolver(), runtimeShadedJarFactory, currentGradleInstallation))
        .invalidNotationMessage("Comprehensive documentation on dependency notations is available in DSL reference for DependencyHandler type.").toComposite();
    }
}
```

从里面我们看到了FileCollection，Project，ClassPathNotation三个类，是不是感觉和我们的三种三方库资源形式很对应？其实这三种资源形式的解析就是用这三个类进行的。DependencyNotationParser就是整合了这些转换器，成为一个综合的转换器。其中，
- DependencyFilesNotationConverter将FileCollection解析为SelfResolvingDependency，也就是implementation fileTree(include: ['*.jar'], dir: 'libs')这种形式。
- DependencyProjectNotationConverter将Project解析为ProjectDependency。也就是implementation project(‘:projectA’)
- DependencyClassPathNotationConverter将ClassPathNotation转成SelfResolvingDependency。也就是implementation ‘xxx’这种。

这三种方式具体的解析方法大家可以自行阅读源码，不是本文重点。所以除了Project会被解析为ProjectDependency以外，其他的都是SelfResolvingDependency。其实ProjectDependency是SelfResolvingDependency的子类。他们的关系可以从SelfResolvingDependency的代码注释中看出。

![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/20190904101030.png)


#### 2.5 ProjectDependency
接下来讲讲ProjectDependency.一个常见的Project引用如下：
```groovy
implementation project(‘:projectA’)
```
这里的implementation我们已经知道是插件添加的扩展，不是gradle自带的。那project呢？这个就是gradle自带的了。delegate是DependencyHandler的project方法。
```java
    // DefaultDependencyHandler.java
    public Dependency project(Map<String, ?> notation) {
        return this.dependencyFactory.createProjectDependencyFromMap(this.projectFinder, notation);
    }

    // DefaultDependencyFactory.java
    public ProjectDependency createProjectDependencyFromMap(ProjectFinder projectFinder, Map<? extends String, ? extends Object> map) {
        return this.projectDependencyFactory.createFromMap(projectFinder, map);
    }

    // ProjectDependencyFactory.java
    public ProjectDependency createFromMap(ProjectFinder projectFinder, Map<? extends String, ?> map) {
        return (ProjectDependency)NotationParserBuilder.toType(ProjectDependency.class).converter(new ProjectDependencyFactory.ProjectDependencyMapNotationConverter(projectFinder, this.factory)).toComposite().parseNotation(map);
    }

    // ProjectDependencyMapNotationConverter.java
    protected ProjectDependency parseMap(@MapKey("path") String path, @Optional @MapKey("configuration") String configuration) {
            return this.factory.create(this.projectFinder.getProject(path), configuration);
    }

    // DefaultProjectDependencyFactory.java
    public ProjectDependency create(ProjectInternal project, String configuration) {
        DefaultProjectDependency projectDependency = (DefaultProjectDependency)this.instantiator.newInstance(DefaultProjectDependency.class, new Object[]{project, configuration, this.projectAccessListener, this.buildProjectDependencies});
        projectDependency.setAttributesFactory(this.attributesFactory);
        projectDependency.setCapabilityNotationParser(this.capabilityNotationParser);
        return projectDependency;
    }
```
我们可以看到，传入的project最终传递给了ProjectDependencyMapNotationConverter。先去查找这个project，然后通过factory去create ProjectDependency对象，当然这里也有考虑到Configuration的影响, 最终是产生了一个DefaultProjectDependency。这就是ProjectDependency的产生过程。


#### 2.6 依赖的体现
看到这里，大家可以已经理解了不同的依赖的解析方式，但是可能还是不理解依赖到底是一个什么东西。**其实依赖库并不是依赖三方库的源代码，而是依赖三方库的产物，产物又是通过一系列的Task执行产生的**。也就是说，projectA依赖projectB，那么A就拥有了对于B的产物的所有权。关于产物我们等后面再介绍。先了解一下Configuration对于产物有一些什么支持。
我们看Configuration的代码， 可以发现他继承了FileCollection接口，而FileCollection又继承了Buildable接口。这个接口有啥用呢，用处大得很。先看[官网](https://docs.gradle.org/current/dsl/org.gradle.api.Buildable.html)介绍

![](https://raw.githubusercontent.com/oubindo/ImageBed/master/img/20190905001835.png)

Buildable表示着很多Task对象生成的产物。它里面只有一个方法。
```java
public interface Buildable {
    TaskDependency getBuildDependencies();
}
```
我们看看DefaultProjectDependency的实现。
```java
    // DefaultProjectDependency.java
    public TaskDependencyInternal getBuildDependencies() {
        return new DefaultProjectDependency.TaskDependencyImpl();
    }

    private class TaskDependencyImpl extends AbstractTaskDependency {
        private TaskDependencyImpl() {
        }

        public void visitDependencies(TaskDependencyResolveContext context) {
            if (DefaultProjectDependency.this.buildProjectDependencies) {
                DefaultProjectDependency.this.projectAccessListener.beforeResolvingProjectDependency(DefaultProjectDependency.this.dependencyProject);
                Configuration configuration = DefaultProjectDependency.this.findProjectConfiguration();
                context.add(configuration);
                context.add(configuration.getAllArtifacts());
            }
        }
    }

    public Configuration findProjectConfiguration() {
        ConfigurationContainer dependencyConfigurations = this.getDependencyProject().getConfigurations();
        String declaredConfiguration = this.getTargetConfiguration();
        Configuration selectedConfiguration = dependencyConfigurations.getByName((String)GUtil.elvis(declaredConfiguration, "default"));
        if (!selectedConfiguration.isCanBeConsumed()) {
            throw new ConfigurationNotConsumableException(this.dependencyProject.getDisplayName(), selectedConfiguration.getName());
        } else {
            return selectedConfiguration;
        }
    }
```
我们可以看到，其实就是在解析每个依赖的时候，如果指定了ConfigurationContainer中声明好的Configuration，比如implementation, api等，那就返回这个Configuration，否则就返回default。拿到这个Configuration之后，做了这个操作
```java
context.add(configuration);
context.add(configuration.getAllArtifacts());
```
这里的context是一个TaskDependencyResolveContext，它的add方法可以添加能contribute tasks to the result的对象，比如Task，TaskDependencies，Buildable等，这些类型都能为产生结果贡献Task(前面我们说到了就是靠Task产生产物的嘛)。 

这里context把configuration和configuration.getAllArtifacts()加入，都是作为Buildable而加入。区别是configuration.getAllArtifacts()获取的是DefaultPublishArtifactSet对象。接下来看看DefaultPublishArtifactSet是怎么实现Buildable的方法的。
```java
    public TaskDependency getBuildDependencies() {
        return this.builtBy;   // 这里的builtBy是下面的ArtifactsTaskDependency对象
    }

    private class ArtifactsTaskDependency extends AbstractTaskDependency {
        private ArtifactsTaskDependency() {
        }

        public void visitDependencies(TaskDependencyResolveContext context) {
            Iterator var2 = DefaultPublishArtifactSet.this.iterator();
            
            while(var2.hasNext()) {
                PublishArtifact publishArtifact = (PublishArtifact)var2.next();
                context.add(publishArtifact);
            }

        }
    }

```
我们发现果然和DefaultPublishArtifactSet的名字一样，是作为set把里面包含的PublishArtifact对象逐个的放入context中。

#### 2.7 总结
我们在这一小节中分析了不同的依赖方式的区别，告诉了大家Configuration是什么东西，也告诉了大家依赖到底是怎么产生和起作用的。这样大家在日常开发中就更能知其所以然了。总结一下就是说
- implementation等都是通过methodmissing机制，被插件解析成不同的Configuration，所以要预定义。
- 我们不同的依赖声明，将会被不同的转化器Parser进行转换。project依赖会被转换为ProjectDependency，其余的会被解析成可自解析的SelfResolvingDependencies。
- project依赖最终是Task和产物artifact的依赖。

那么Task和产物又是一种什么关系呢？这个就涉及到更深层次了。本文篇幅有限，放在下一篇再分析吧。


## 总结
本文主要是从Extension和Plugin引入，主要讲解了gradle依赖的原理和解析机制，然后抛下了一个疑问：产物artifacts是怎么和依赖扯上关系的呢？这个问题我们等下一篇《一篇文章深入gradle（下篇）:Artifacts》再解答






---
title: gradle学习
date: 2018-08-09 17:08:29
tags: android
categories: 
---

## gradle学习之路
1.gradle init生成gradle需要的文件。

2.gradle task --scan 可以生成报表，方便我们查看和分析什么耗时

3.build.gradle 里面写的是项目多模块可以同时使用的配置。当我们在这里加上
```
allprojects {
    repositories {
        jcenter() 
    }
}
```
的时候，配置就对全局有效。也可以改成subprojects。而且这两个配置可以在根项目里使用很多次。

4.生成library： gradle init --type groovy-library/java-library/java-application

5.多个模块的通用设置放在根目录的build.gradle下面。可以用configure语句
```
configure(subprojects.findAll { it.name == 'greeter' || it.name == 'greeting-library' }) { 

    apply plugin: 'groovy'

    dependencies {
        testCompile 'org.spockframework:spock-core:1.0-groovy-2.4', {
            exclude module: 'groovy-all'
        }
    }
}
```

6.只在根目录，进行其他subproject的编译
./gradlew :{subproject}/{task}

7.在Java library中进行build，可以在build/libs目录下找到jar。或者也可以使用./gradlew jar命令打jar包



---
layout: post
#标题配置
title:  找不到或无法加载主类
#时间配置
date:   2017-11-01 16:16:00 +0800
#大类配置
categories: Java
#小类配置
tag: 
---

* content
{:toc}

# 1. 背景
近来学习分布式系统，对于远程调用的学习结合实例RMI,遇到使用命令行编译运行java program,结果出现了一些问题
有关于javac带package（含包组织的）的.java, 以及如何运行含有包组织的类文件，还有关于rmiregistry去自动加载类的问题

学了java这么多年，很少使用命令行运行java，说来真是惭愧。。。。。。

记录解决“错误: 找不到或无法加载主类”， 记录学习java命令行运行含包组织的java工程

# 2. javac and java the file of .java
## 2.1 工程目录结构
```buildoutcfg
～/projects/offiRmi/src/              (工程源代码目录)
                       com.example.   (包结构com.example)
                                   Test.java  (java文件)
                      
```

## 2.2 javac编译源代码
```buildoutcfg
javac -d ~/projects/rmi com/example/Test.java

note:
    从包结构外部进行编译，com/的父目录及以上
    
    -d destDir 表示编译后的.class文件存放的地方
```

## 2.3 java运行.class文件
```buildoutcfg
the cases of  success:

在非.class文件目录下，在源文件src下
lyman@lyman:~/projects/offiRmi/src$ java -classpath ~/projects/rmi com/example/Test
lyman@lyman:~/projects/offiRmi/src$ java -classpath ~/projects/rmi com.example.Test

在.class文件目录下
lyman@lyman:~/projects/rmi$ java com.example.Test



the cases of failure:

在非.class目录下
lyman@lyman:~/projects/offiRmi/src$ java ~/projects/rmi/com/example/Test
错误: 找不到或无法加载主类 .home.lyman.projects.rmi.com.example.Test

在.class文件目录下
lyman@lyman:~/projects/rmi$ cd com
lyman@lyman:~/projects/rmi/com$ java example.Test 
错误: 找不到或无法加载主类 example.Test
lyman@lyman:~/projects/rmi/com$ cd example/
lyman@lyman:~/projects/rmi/com/example$ java Test
错误: 找不到或无法加载主类 Test

```
Note:说明对于使用java package的文件，不管是在.class文件目录下运行还是在非.class文件下运行，
都需要在package的外部完整的调用 包名.二级包名....类名，来运行class文件
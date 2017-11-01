---
layout: post
#标题配置
title:  Distributed System(2)--rmi简单Hello world
#时间配置
date:   2017-11-01 16:58:00 +0800
#大类配置
categories: Distributed System
#小类配置
tag: 
---

* content
{:toc}

# 1. 背景
在学习分布式系统中，打算使用rpc的常见实例RMI，做一些东西。在测试环境运行简单Hello world出现了两个错误

1. 错误: 找不到或无法加载主类  这个在java中的load class中记录解决
2. java.lang.ClassNotFoundException: com.example.Hello  无法加载远程对象Hello

本文记录问题2

[Getting Started Using JavaTM RMI](https://docs.oracle.com/javase/6/docs/technotes/guides/rmi/hello/hello-world.html)

# 2. ClassNotFoundException: com.example.Hello
折腾过后找到的原因是，启动rmiregistry &服务的时候，没有在 .class的包顶层的根目录下执行该命令；在根目录下执行rmiregristy,正常完成

```buildoutcfg
1. 编译程序源文件
lyman@lyman:~/projects/offiRmi/src$ ls com/example/
Client.java  Hello.java  Server.java  Test.java
lyman@lyman:~/projects/offiRmi/src$ javac -d ~/projects/rmi com/example/*.java

2. 开启rmiregistry进程，note: 是在.class文件的顶层目录下执行
lyman@lyman:~/projects/rmi$ rmiregistry &
[1] 10337

3. run the class
lyman@lyman:~/projects/offiRmi/src$ java -classpath ~/projects/rmi -Djava.rmi.server.codebase=file:~/projects/rmi/ com.example.Server &
lyman@lyman:~/projects/offiRmi/src$ com.example.Server ready

Note: 在run the class中，运行的自然是.class，需要指定-Djava.rmi.server.codebase用来加载类信息，最后的‘/’不要丢掉

```

# 3. RMI如何做到远程调用的
## 3.1 Server class
'Server.class' 要做的事儿：

1. create an instance of  the remote object implementation, and export the remote object
2. bind that instance to a name in a Java RMI registry
3. 实例化 remote interface 中定义的方法

note: 目前Server自身的main()中实现了这些逻辑，也可以用另外的类实现； 只有在remote interface 中定义的方法采用远程调用，在非remote interface中声明的方法只能在同一个jvm上

```buildoutcfg
Server obj = new Server();
Hello stub = (Hello) UnicastRemoteObject.exportObject(obj, 0);
```
UnicastRemoteObject.exportObject(obj, 0) 做了两件事儿

1. 使本地的runtime listen/accept incoming remote calls
2. **return the stub for the remote object to pass to client**, stub实现了同样的远程接口，并且包含server的host name and port, 以便远程定位and连接

现在有了remote interface的实现和stub to pass to client, 接下来要做的是如何让client找到stub并且使用该stub

```buildoutcfg
Registry registry = LocateRegistry.getRegistry();
registry.bind("Hello", stub);
```
通过使用Registry bind a name to a remote object's stub, 使得client在lookup(name)的时候可以obtain stub.
---
layout: post
#标题配置
title:  Distributed System(1)--大端小端&rpc
#时间配置
date:   2017-10-13 15:05:00 +0800
#大类配置
categories: Distributed System
#小类配置
tag: 
---

* content
{:toc}

# 1. 参数传递
## 1.1 小端大端模式
计算机存储中使用字节作为基本的单位，而很多变量的类型包含很多字节，所以需要考虑将这么几个意义连续的字节如何存放在一段连续的地址中

大端：数据的高字节存放到存储空间的低地址中
小端：数据的高字节存储到存储空间的高地址中

对教材中值传递图片例子的理解：
![大端小端模式]({{'/styles/images/Distribute/2017-10-14-distributed-system-1-01.png' | prepend: site.baseurl }})
图片表示通信过程中对大小端无要求的从服务器a　传递参数到服务器b　的过程，
a机器采用小端存储对于0x0005,将数据的高字节放到存储空间的高字节中
需要注意的是在网络通信的过程中，数据是按照字节传输的（多个位信息），b是a把数据传输过去，在b上按照字节接受的顺序进行存储。
（可以看到a,b中信息存储位置一样）而由于b机器采用大端模式，也就是数据的高字节存储到存储空间的低地址中，所以将前４个字节的信息取出来
得到的是0x5000(5*2^24)，出现了数据不一致的问题！！！！

而对于字符串则没有影响，因为字符串是使用char[]数组进行存储的，char类型的数据只占一个字节，没有字节的排列顺序问题

## 1.2 网络数据传输场景
通过上面例子可以看到，在网络上传输数据时，由于数据传输两端的机器可能使用不同的存储模式。所以TCP/IP规定在网络上传输采用网络字节顺序，也就是big-endian大端模式。
对于非char类型的数据，在发送前需要转化成大端模式，接受网络数据时按主机环境接受。

# 2. 通信
## 2.1 RPC
### 2.1.1 what's rpc
* A type of client/server communication
* To make remote procedure call look like local ones

当机器A的进程调用机器B的进程时，A的调用过程被挂起，并将参数通过信息传递给B，B上的进程开始执行，执行完成后传回结果

### 2.2.2 the goals of rpc
1. hide complexity
2. automates task of implementing distributed computaiton
3. makes a call to remote service look like a local call
   * transparent in the server's location local or remote
   * transparent in the process of distributeing applications
   * transparent in the architecture of remote machine
   
### 2.2.3 the problems of rpc
1. 机器环境的异构性，operating systems, 编码形式。。
2. 将传输的data进行本地化的表示
3. 调用消息在网络传输的不确定性，网络异常

### 2.2.4 how to obatain transparency
using API stubs on the client and server ,(存根)
1. Client stub

* marshalling arguments(将参数编码成和底层机器无关的形式)
* send request to server
* wait for responce
* unmarshalling result, and retuuns to caller

2. Server stub

* unmarshalling arguments and builds stack frame
* call procedure
* marshalling results and send reply

在client/server stub的执行过程过多次提到marshalling,unmarshalling,通过使用的统一的网络编码模式，完成底层异构机器之间的网络信息传输，
在网络传输中使用标准的大端模式，big-endian

### 2.2.5 how to write stubs
using an IDL -- interface definition language

# 3. The process of  Remote Procedure Calls
Client:

1. The client procedure calls the client stub in the normal way.
2. The client stub builds a message and calls the local operating system.
3. The client’s OS sends the message to the remote OS.

Server:

1. The remote OS gives the message to the server stub.
2. The server stub unpacks the parameters and calls the server.
3. The server does the work and returns the result to the stub.
4. The server stub packs it in a message and calls its local OS.
5. The server’s OS sends the message to the client’s OS.
6. The client’s OS gives the message to the client stub.

Client:

1. The stub unpacks the result and returns to the client.

![通过RPC的远程计算]({{'/styles/images/Distribute/2017-10-14-distributed-system-1-02.png' | prepend: site.baseurl }})
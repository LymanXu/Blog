---
layout: post
#标题配置
title:  Common install
#时间配置
date:   2017-11-14 16:40:00 +0800
#大类配置
categories: 
#小类配置
tag: 
---

* content
{:toc}

# 1. Jdk安装配置
```buildoutcfg
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.tar.gz
后面的链接按照版本替换
tar -zxvf jdk-8u151-linux-x64.tar.gz
mv jdk1.8.0_151 /opt/jdk

vim /etc/profile
加入：
export JAVA_HOME=/opt/jdk1.8
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export HADOOP_HOME=/opt/hadoop2.7
export PATH=.:${JAVA_HOME}/bin:${HADOOP_HOME}/bin:$PATH

生效服务
source /etc/profile  

```

# 2. Jdk卸载
sudo apt autoremove openjdk*
whereis java 把对应目录删除

# 3. Hadoop配置
```buildoutcfg
core-site.xml  

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>  
       <name>hadoop.tmp.dir</name>  
       <value>file:/home/lyman/tmp</value>  
    </property>    
</configuration>  


hdfs-site.xml  

<configuration>  
   <property>   
       <name>dfs.replication</name>   
       <value>1</value>   
   </property>   
   <property>   
       <name>dfs.namenode.name.dir</name>   
       <value>file:/home/lyman/tmp/dfs/name</value>   
   </property>   
   <property>   
       <name>dfs.datanode.data.dir</name>   
       <value>file:/home/lyman/tmp/dfs/data</value>   
   </property>  
</configuration>  


mapred-site.xml(没有则复制一份mapred-site.xml.template并命名为mapred-site.xml)  

<configuration>  
<property>     
       <name>mapreduce.framework.name</name>   
       <value>yarn</value>      
</property>  
</configuration>  


yarn-site.xml  

<configuration>  
<property>  
       <name>yarn.nodemanager.aux-services</name>  
       <value>mapreduce_shuffle</value>  
</property>  
<property>  
       <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>  
       <value>org.apache.hadoop.mapred.ShuffleHandler</value>  
</property>  
</configuration> 
```

---
layout: post
#标题配置
title:  Tricircle开发问题List
#时间配置
date:   2017-07-05 15:17:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 教程
---

* content
{:toc}

# 1. 问题List
## 1.1 import fixtures error
没有该种模块，进行pip安装到该项目下
```buildoutcfg
sudo pip install fixtures
```
## 1.2 远程桌面
通过安装Tigervnc可以从远程访问显示桌面
1. download vnc : wget https://bintray.com/tigervnc/stable/download_file?file_path=tigervnc-1.7.1.x86_64.tar.gz.
2. tar -xzvf xxx
3. 进入目录tigervnc-1.7.1.x86_64/usr,然后用sudo权限将所有文件拷贝到/usr/目录下.命令"sudo cp -r * /usr/".
4. 用"vncserver :1"开启一个远程登陆的端口


# 2. Test问题
Python是个动态语言，很多问题都无法通过静态编译检查来发现，单元测试成了一个重要的确保质量的手段<br/>
[**tox单元测试过程&如何配置**](https://blog.apporc.org/2016/08/python-%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7-tox/)<br/>
[**单元测试工具介绍**](http://www.tuicool.com/articles/UnQbyyv)<br/>
tox 用来管理和构建不同类型单元测试所需要的环境，如py27依赖一些库的环境，py35,pypy
## 2.1 Tox测试问题
1. Pypy没有安装

```buildoutcfg
sudo add-apt-repository ppa:pypy/ppa
sudo apt-get update
(或直接运行安装)
sudo apt-get install pypy pypy-dev
```
2. Pypy: could not install deps

[参考](https://github.com/GoogleCloudPlatform/google-cloud-python/issues/1103)
```buildoutcfg
$ sudo /usr/bin/pip install tox --upgrade
```

## 2.2 tricircle在线部署测试
1. 将开发代码cp到tricircle节点的 /opt/stack/tricircle

    * 将节点上的目录更改备份.mv 既可以重命名目录又可以移动文件和文件夹, mv olddir newdir(将旧的目录重新命名), mv a b/c(将目录a 移动到b下并重命名为c)
    * 将本地的文件上传到虚拟机. scp -r ~/PycharmProjects/tricircle stack@192.168.56.102:~/testmv,将tricircle整个文件夹直接copy
    到目的目录~/testmv下（形成~/testmv/tricircle）;对于目录重命名scp -r ~/PycharmProjects/tricircle/* stack@192.168.56.102:~/testmv/tricirclenew
    
2. 重启服务  
    * 重启apache服务：sudo /etc/init.d/apache2 restart (对于某些服务重启appach服务不一定生效，例如xjob)
    * 目前openstack使用systemctl进行服务的管理，[Using Systemd in DevStack](https://docs.openstack.org/devstack/latest/systemd.html)
    [systemctl的使用](http://blog.csdn.net/u012486840/article/details/53161574)

  对于这种服务的更新，可以使用　ps -aux | grep xjob查看该服务对应的进程，通过kill 进程再次重启进程
3. source openrc admin admin 
4. 获取token(openstack token issue), token=$(openstack --os-region-name CentralRegion token issue | awk 'NR==5 {print $4}')
5. 测试代码　curl -X GET http://127.0.0.1/tricircle/v1.0/jobs?status=new  -H "Content-Type: application/json"  -H "X-Auth-Token: $token"　

　　多个参数时　jobs?status=new\&resource_type=port
   
   安装json格式化，sudo apt-get install jp<br/>
   查看结果在curl 语句末尾加上　| jp
   
对于openstack中的某些命令不知道可以使用　openstack help | grep security
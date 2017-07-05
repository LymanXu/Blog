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
没有该种模块，进行sudo pip install fixtures

## 1.2 依赖原有分支进行修改
1. create a floder, and run "git init"
2. configure remote reposity, 添加远程源 "git remote add gerrit https://Launchpad_id@review.openstack.org/openstack/tricircle.git"
3. download the remote patch. run "git review -d patch_number".
4. 完成该分支的下拉同步，进行修改
5.1 在原有分支上进行更改

   ‘git add xxx’<br/>
   'git commit -a --amend'<br/>
   'git review'<br/>
5.2 新建分支依赖原有分支进行修改
   'checkout new branch'<br/>
   'git add xxx'<br/>
   'git commit'新的分支上首次进行修改，之后使用-a --amend<br/>
   'git review'<br/>
   
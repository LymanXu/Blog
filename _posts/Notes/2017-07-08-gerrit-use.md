---
layout: post
#标题配置
title:  Gerrit在Tricircle开发中的使用总结
#时间配置
date:   2017-07-08 11:22:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 教程
---

* content
{:toc}

# 1. 依赖分支的场景
## 1.1 基于原有分支进行修改
**Question**<br/>
在写tricircle的job paginate逻辑的时候，我需要依赖正在开发的routing_list分支(其中提供了底层core对util.paginate调用的接口)，
要做的是基于routeing_list新建一个分支job_paginate，并在上面完成job paginate

1. create a floder, and run "git init"
2. configure remote reposity, 添加远程源 "git remote add gerrit https://Launchpad_id@review.openstack.org/openstack/tricircle.git"
3. download the remote patch. run "git review -d patch_number".
4. 完成该分支的下拉同步，进行修改
5. 在原有分支上进行更改
   
   ‘git add xxx’<br/>
   'git commit -a --amend'<br/>
   'git review'<br/>
   
6. 新建分支依赖原有分支进行修改
   
   'checkout new branch'<br/>
   'git add xxx'<br/>
   'git commit'新的分支上首次进行修改，之后使用-a --amend<br/>
   'git review'<br/>
   
## 1.2 原有分支发生新的更新，同步依赖分支工作到我的分支
**Question**<br/>
原有依赖的分支routing_list上做了一些更新，提交，这意味着routing_list和job_paginate两个分支各自前进了，之间分叉了，在这里需要将原始依赖分支routing_list拉取下来，同步到job_paginate分支上

1. create a floder, run 'git init'
2. config remote reposity, 'git remote add gerrit https://Launchpad_id@review.openstack.org/openstack/tricircle.git'
3. 下载job_paginate分支， 'git review -d patch_number'
4. 目前是在job_paginate分支上，需要将依赖分支的更改拉取下来， 'git review -d $path_change_number'(注意这个是一个开发分支的changeID,不是path Number)

   这一步会将分支切换到routing_list
5. 'git review -x $child_change_number'(注意$child_change_number是子分支的changeID,job_paginate的changeID,而不是path NUmber)
6. 重新命名当前分支， ’git branch -m job_paginate_new‘
7. 'git commit -a --amend'(无新的更改则不需要)
8. 'git review'(会提交到job_paginate远程上)



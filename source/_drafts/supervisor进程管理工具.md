---
title: supervisor进程管理工具
tags:
  - supervisor
  - 进程管理
categories:
  - 运维
---

## 问题描述
安装pipenv时使用sudo python3 -m pip install pipenv 
在supervisor中直接运行pipenv ... 时其将virtualenv定位到/root/.local/share/virtualenv文件夹下，导致没有足够的权限进行访问

## 解决方案
将当前项目的虚拟运行环境安装在当前项目下，即添加环境变量　　使得将虚拟环境安装在项目目录
然后按照正常方式运行即可


---
bg: "tree.jpeg"
layout: post
title: "Simple Python Version Management: pyenv"
summary: ""
tags: ['python']
categories: python
---

python有2.\*和3.\* 版本，通常系统自带2.\*，然而python官方后续不再继续维护2.\*版本, 而且像asyncio这种异步官网库也只有在3.*上才有。所以通常情况下我们会安装多个版本，然后通过类似python2.6，python2.7，python3这样子来区分不同的python版本，有一个问题就是如果需要脚本需要x权限时，需要在文件开始添加/path/2.\*这种命令，不利于开发环境往生产环境移植，而且如果在多用户系统下，不能随便修改python命令的名称，因为不确定会否对别人造成影响

[pyenv](https://github.com/pyenv/pyenv) 是一个简单的python版本管理工具，它可以使当前用户的python版本一键切换，关键是不影响其他用户，这样我们想要执行python解释器或python命令时，只需要切换到对应版本，然后统一用python，而不需要输入python2.7或python3.6, 从而也避免了线上线下环境发布时对python版本名称不一致的问题


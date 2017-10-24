---
bg: "programer.jpeg"
layout: post
title: '[转] for() && fork() || fork()'
summary: ""
tags: ['coding']
categories: coding
---

[原文](http://blog.csdn.net/lzm18064126848/article/details/62222497)

```c
#include <unistd.h>  
#include <stdio.h>  
int main()  
{  
        fork();/*****/  
        fork() && fork() || fork();/*****/  
        fork();/*****/  
        sleep(100);  
        return 0;  
}  
```

问题是不算main这个进程自身，程序到底创建了多少个进程?

这是EMC的一道笔试题，感觉挺有意思的，这道题主要考了两个知识点，一是逻辑运算符运行的特点；二是对fork的理解
如果有一个这样的表达式：cond1 && cond2 || cond3 这句代码会怎样执行呢？
1、cond1为假，那就不判断cond2了，接着判断cond3
2、cond1为真，这又要分为两种情况：
  a、cond2为真，这就不需要判断cond3了
  b、cond2为假，那还得判断cond3
fork调用的一个奇妙之处在于它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：
1、在父进程中，fork返回新创建子进程的进程ID；
2、在子进程中，fork返回0；
3、如果出现错误，fork返回一个负值（题干中说明了不用考虑这种情况）
在fork函数执行完毕后，如果创建新进程成功，则出现两个进程，一个是子进程，一个是父进程。在子进程中，fork函数返回0，在父进程中，fork返回新创建子进程的进程ID。我们可以通过fork返回的值来判断当前进程是子进程还是父进程。
有了上面的知识之后，下面我们来分析fork() &&　fork() || fork()会创建几个新进程

![fork](/assets/images/posts/20171024/fork.jpg)



很明显fork() &&　fork() || fork()创建了4个新进程

总结：

第一注释行的fork生成1个新进程
第二注释行的三个fork生成4+4=8个新进程
第三注释行的ork会生成10个新进程(这是因为前面总共有10个进程,调用一次fork生成10个新进程)

所以一共会生成1+8+10=19个新进程
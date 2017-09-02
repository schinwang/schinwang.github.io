---
bg: "tree.jpeg"
layout: post
title: ".bashrc与.bash_profile区别"
summary: ""
tags: ['notes']
categories: notes
---

采用bashshell进行交互操作时，在终端当前用户的home目录下经常可以看到.bashrc, .bash_profile 在/etc/目录下也会看到/etc/bashrc, /etc/profile.

关于.bashrc与.bash_profile的区别，stack上已经有了比较明朗的回答:[whats the difference between bashrc bash profile and environment](https://stackoverflow.com/questions/415403/whats-the-difference-between-bashrc-bash-profile-and-environment)

执行man bash命令也可以看到：

>The following paragraphs describe how bash executes its startup  files. If  any  of  the files exist but cannot be read, bash reports an error.  Tildes are expanded in file names as described below under Tilde Expansion in the EXPANSION section.
>
>When  bash is invoked as an interactive login shell, or as a non-inter‐       active shell with the --login option, it first reads and executes  commands  from  the file /etc/profile, if that file exists.  After reading that file, it looks for ~/.bash_profile, ~/.bash_login, and ~/.profile, in  that order, and reads and executes commands from the first one that exists and is readable.  The --noprofile option may be  used  when  the shell is started to inhibit this behavior.
>
>如果bash以交互登录或非交互登录模式唤起时，它会先读取并执行/etc/profile(如果存在的话)，然后以先后顺序执行~/.bash_profile, ~/.bash_login, ~/.profile. 如果任何文件不可读，bash都会报错
>
>When  a  login  shell  exits, bash reads and executes commands from the files ~/.bash_logout and /etc/bash.bash_logout, if the files exists. When an interactive shell that is not a login shell  is  started,  bash reads  and executes commands from ~/.bashrc, if that file exists.  This may be inhibited by using the --norc option.  The --rcfile file  option will  force  bash  to  read  and  execute commands from file instead of ~/.bashrc.
>
>如果bash以非登录方式唤起（比如su user切换到某个用户），bash会读取并执行~/.bashrc
>
>When bash is started non-interactively, to  run  a  shell  script,  for example, it looks for the variable BASH_ENV in the environment, expands
>
>its value if it appears there, and uses the expanded value as the  name
>
>of  a  file to read and execute.  Bash behaves as if the following command were executed:
>
>​              if [ -n "\$BASH_ENV" ]; then . "$BASH_ENV"; fi
>
>but the value of the PATH variable is not used to search for  the  file name.
>
>如果bash以非交互方式启动（比如执行某个脚本文件），它会搜索BASH_ENV变量，如果存在, 则会将其值扩展为一个文件名并执行它
>
>If  bash  is  invoked  with  the name sh, it tries to mimic the startup behavior of historical versions of sh as  closely  as  possible,  while conforming  to the POSIX standard as well.  When invoked as an interac‐tive login shell, or a non-interactive shell with the  --login  option, it  first  attempts  to read and execute commands from /etc/profile and ~/.profile, in that order.  The  --noprofile  option  may  be  used  to inhibit  this behavior.  When invoked as an interactive shell with the name sh, bash looks for the variable ENV, expands its value  if it is defined,  and uses the expanded value as the name of a file to read and execute.  Since a shell invoked as sh does not attempt to read and execute  commands from any other startup files, the --rcfile option has no effect.  A non-interactive shell invoked with  the  name  sh  does  not attempt  to  read  any  other  startup files.  When invoked as sh, bash enters posix mode after the startup files are read.
>
>如果bash以sh命令的方式唤起，它会在遵从POSIX标准的同时，尽可能模拟之前的启动行为. 如果是以交互（或非交互）登录方式唤起，它会先后读取并执行/etc/profile和~/.profile. 如果sh以交互(非登录)方式唤起bash,它会寻找ENV变量，扩展其值为文件名并执行该文件(如果存在且可读)
>
>When bash is started in posix mode, as with the  --posix  command  line       option, it follows the POSIX standard for startup files.  In this mode,      interactive shells expand the ENV variable and commands  are  read  and executed  from  the  file  whose  name is the expanded value.  No other startup files are read.
>Bash attempts to determine when it is being run with its standard input      connected to a network connection, as when executed by the remote shell daemon, usually rshd, or the secure shell daemon sshd.  If bash  deter‐       mines  it  is being run in this fashion, it reads and executes commands from ~/.bashrc, if that file exists and is readable.  It  will  not  do this  if  invoked as sh.  The --norc option may be used to inhibit this behavior, and the --rcfile option may be used to force another file  to be  read,  but  rshd  does  not  generally  invoke the shell with those options or allow them to be specified.
>
>Bash会尝试根据输入决定它是否在以网络连接或shell daemon(rshd or sshd)执行的方式在运行，如果是的话，它会读取并执行~/.bashrc文件
>
>If the shell is started with the effective user (group) id not equal to the real user (group) id, and the -p option is not supplied, no startup files are read, shell functions are not inherited from the environment, the  SHELLOPTS,  BASHOPTS,  CDPATH,  and  GLOBIGNORE variables, if they appear in the environment, are ignored, and the effective  user id is set  to  the real user id.  If the -p option is supplied at invocation, the startup behavior is the same, but the  effective  user  id  is  not reset.

综上

* 以登录方式唤起bash时，会执行/etc/profile, .bash_profile文件
* 以非登录交互方式启动时，会执行.bashrc文件
* ssh 远程执行命令时会，会执行.bashrc文件
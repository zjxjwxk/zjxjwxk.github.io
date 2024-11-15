---
title: CentOS jar包后台运行
date: 2019-01-30 00:54:00
tags: 
- Linux
- CentOS
- Java
categories: 
- Linux
preview: 50
---

# 运行jar包命令

```shell
java -jar application.jar
```

当前ssh窗口被锁定，可按CTRL + C打断程序运行，或直接关闭窗口让程序退出。

那如何让窗口不被锁定？

# 方法一

```shell
java -jar application.jar &
```

&代表在后台运行。

当前ssh窗口不被锁定，但是当窗口关闭时，程序中止运行。

继续改进，如何让窗口关闭时，程序仍然运行？

# 方法二

```shell
nohup java -jar application.jar &
```

nohup 意思是不挂断运行命令,当账户退出或终端关闭时,程序仍然运行。

那怎么不打印该程序输出的内容，而写入某个文件中呢？

# 方法四

```shell
nohup java -jar application.jar >out.txt &
```

当用 nohup 命令执行作业时，缺省情况下该作业的所有输出被重定向到nohup.out的文件中，除非另外指定了输出文件，如 out.txt。

command >out.txt是将command的输出重定向到out.txt文件，即输出内容不打印到屏幕上，而是输出到out.txt文件中。

### 通过jobs命令查看后台运行任务

```shell
jobs
```

那么就会列出所有后台执行的作业，并且每个作业前面都有个编号。

### 如果想将某个作业调回前台控制，只需要 fg + 编号即可。

```shell
fg 23
```

### 查看某端口占用的线程的pid

```shell
netstat -nlp |grep :9181
```


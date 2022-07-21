---
title: "MIT_6.S081_lab1：Xv6 and Unix utilities"
date: 2022-06-27T00:19:52+08:00
lastmod: 2022-06-27T00:19:52+08:00
author: ["Idealist"]
tags:
- Lab
- OS
- MIT 6.S081
mermaid: false #是否开启mermaid
# summary->在列表页展现的摘要内容，自动生成，内容默认前70个字符，可通过此参数自定义，一般无需专门设置
summary: ""
# description->需要自己编写的文章描述，是搜索引擎呈现在搜索结果链接下方的网页简介，建议设置
description: "MIT 6.S081 实验一 笔记"
weight: # 输入1可以置顶文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true # 顶部显示当前路径
cover:
    image: "https://unix.org/images/unix-an-open-group-standard.png"
    caption: ""
    alt: ""
    relative: false
---



## 0. 引言

在跟随《操作系统真象还原》实现了一个 mini os 的大部分功能后，我选择刷一遍 MIT 的这门课，并将 lab 的解题过程整理成文，记录到博客上供自己总结和网友参考。

> 课程地址：<https://pdos.csail.mit.edu/6.S081/2020/schedule.html>
>
> 课程视频翻译：<https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/>
>
> 我的代码地址：<https://github.com/Idealist226/xv6-labs-2020>

## 1. 准备

- 开发环境：VMWare + Ubuntu 20.04

- 根据 **[官方指南](https://pdos.csail.mit.edu/6.828/2020/tools.html)** 中"Installing via APT (Debian/Ubuntu)" 一节安装 qemu 等工具

- 克隆代码到本地仓库，切换分支（`git checkout`）并编译

  ```shell
  git clone git://g.csail.mit.edu/xv6-labs-2020
  cd xv6-labs-2020
  git checkout util
  make qemu
  ```

- 在 GitHub 上新建远程仓库，并在本地代码仓库中添加主机名对应到远程仓库

  ```shell
  git remote add github 你的仓库地址
  cat .git/config
  ```
  
  ![image-20220628002735344](https://cdn.jsdelivr.net/gh/Idealist226/img/imageimage-20220628002735344.png)
  
- 建议：每个实验创建一个测试分支（以 util 为例），测试通过后提交（`git commit`）代码，并将所做的修改合并（`git merge`）到原分支中，然后提交（`git push`）到 GitHub

  ```shell
  git checkout util         # 切换到util分支
  git checkout -b util_test # 建立并切换到util的测试分支
  git add .
  git commit -m "完成了第一个作业"
  git checkout util
  git merge util_test
  git push github util:util
  ```
  

## 2. Lab 1: Xv6 and Unix utilities

> 实验地址：<https://pdos.csail.mit.edu/6.S081/2020/labs/util.html>

This lab will familiarize you with xv6 and its system calls.

熟悉 xv6 与它的系统调用。

### Boot xv6 (easy)

已在`1. 准备`中提及，实现过程请看 Lab 的具体要求。

### sleep (easy)

> Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.

热身题，熟悉一下如何在 6.S081 中写程序。

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
    if (argc <= 1) {
        fprintf(2, "sleep: need one arg for sleep time!\n");
        exit(1);
    }
    sleep(atoi(argv[1]));
    exit(0);
}
```

将 sleep 加入 Makefile 的构建目标中，并进行构建与测试。

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep\
```

![image-20220628011459989](https://cdn.jsdelivr.net/gh/Idealist226/img/imageimage-20220628011459989.png)

### pingpong (easy)

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

在解决这道题之前需要先了解什么是`pipe`。 `pipe` 翻译成中文是“管道”的意思，数据可以从管道的一端输入，从管道的另一端读出，非常形象。其本质上是一片可被读写的内核缓冲区，允许不同的进程借此进行通信。

创建管道的函数原型是`int pipe(int fd[2])`。其接收一个数组，用来存放管道创建完成后使用的两个文件描述符，fd[0]：读管道，fd[1]：写管道。

在这道题中，我创建了两个 `pipe` ：一个用于父进程写数据，子进程读数据；另一个用于父进程读数据，子进程写数据。需要**注意**的是：及时关闭管道无用的读写端。

```c
#include "kernel/types.h"
#include "user/user.h"

void errorHandler(const char* str) {
    fprintf(2, str);
    exit(1);
}

int main() {
    int p2c[2], c2p[2];
    int parentpid, childpid;
    int pid;
    char buf;

    // Create two pipes for communication between parent and child processes
    if (pipe(p2c) == -1 || pipe(c2p) == -1) {
        errorHandler("pipe: create error!\n");
    }
    pid = fork();

    if (pid < 0) {
        errorHandler("fork: create error!\n");
    } else if (pid != 0) {  // parent
        close(p2c[0]);    // p2c write only
        close(c2p[1]);    // c2p read only

        parentpid = getpid();
        write(p2c[1], "g", 1);
        close(p2c[1]);

        read(c2p[0], &buf, 1);
        close(c2p[0]);
        fprintf(1, "%d: received pong\n", parentpid);
    } else {                // child
        close(p2c[1]);    // p2c read only
        close(c2p[0]);    // c2p write only

        childpid = getpid();
        read(p2c[0], &buf, 1);
        close(p2c[0]);

        write(c2p[1], "z", 1);
        close(c2p[1]);
        fprintf(1, "%d: received ping\n", childpid);
    }

    exit(0);
}
```

### primes (moderate)/(hard)

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

利用多线程和管道实现素数的筛选：在每一个进程中，筛掉某个素数的所有倍数。如下图所示：

![sieve](https://cdn.jsdelivr.net/gh/Idealist226/img/imagesieve.gif)

需要**注意**：exit() 是结束当前进程，wait() 是等待子进程结束后才向下执行。

```c
#include "kernel/types.h"
#include "user/user.h"

void sieve(int pleft[]) {
    int p;
    int pright[2];

    close(pleft[1]);     // close pleft write
    if (read(pleft[0], &p, sizeof(int))) {
        fprintf(1, "prime %d\n", p);
        pipe(pright);
        if (fork() != 0) {
            close(pright[0]); // close pright read
            int buf;
            while(read(pleft[0], &buf, sizeof(int))) {
                if (buf % p) {
                    write(pright[1], &buf, sizeof(int));
                }
            }
            close(pleft[0]);  // close pleft read
            close(pright[1]); // close pright write;
            wait(0);
        } else {
            sieve(pright);
        }
    }
    exit(0);
}

int main() {
    int pinput[2];
    pipe(pinput);
    if (fork() != 0) {
        close(pinput[0]);   // close read
        for (int i = 2; i <= 35; i++) {
            write(pinput[1], &i, sizeof(int));
        }
        close(pinput[1]);   // close write
        wait(0);
    } else {
        sieve(pinput);
    }
    exit(0);
}
```

### find (moderate)

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`.

这题基本原理与 `user/ls.c` 大同小异，需要**注意**不要踩坑的是：跳过子目录的 “.” 和 “..” 两个目录文件，否则递归会陷入死循环。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* getFileName(char* path) {
    char *p;
    for (p = path + strlen(path); p >= path && *p != '/'; p--);
    return ++p;
}

void find(char* path, char* target) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0) {
        fprintf(2, "find: connot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type) {
        case T_FILE:
            if (strcmp(getFileName(path), target) == 0) {
                printf("%s\n", path);
            }
            break;
        case T_DIR:
            if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';
            while (read(fd, &de, sizeof(de)) == sizeof(de)) {
                if (de.inum == 0) {
                    continue;
                }
                if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0) {  // skip . and .. of subdirectories
                    continue;
                }
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;
                if (stat(buf, &st) < 0) {
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }
                find(buf, target);  // recursive search
            }
            break;
    }
    close(fd);
}

int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(2, "find: need two args");
        exit(1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```

### xargs (moderate)

> Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file `user/xargs.c`.

编写 xargs 工具，将标准输入中的数据作为额外参数执行。

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

int main(int argc, char* argv[]) {
    int buf_idx, read_len;
    char buf[512];
    char* exe_argv[MAXARG];
     // copy execute args, skip "xargs" at argv[0]
    for (int i = 1; i < argc; i++) {
        exe_argv[i - 1] = argv[i];
    }
    while(1) {
        // read next line char until read '\n' or end
        buf_idx = -1;
        do {
            buf_idx++;
            read_len = read(0, &buf[buf_idx], sizeof(char));
        } while(read_len > 0 && buf[buf_idx] != '\n');
        if (read_len == 0 && buf_idx == 0) {
            break;
        }
        buf[buf_idx] = '\0';
        exe_argv[argc - 1] = buf;

        if (fork() == 0) {
            exec(exe_argv[0], exe_argv);
            exit(0);
        } else {
            wait(0);
        }
    }
    exit(0);
}
```


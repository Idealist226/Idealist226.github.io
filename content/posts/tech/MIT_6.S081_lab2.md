---
title: "MIT_6.S081_lab2: system calls"
date: 2022-07-21T23:35:45+08:00
lastmod: 2022-07-27T21:46:45+08:00
author: ["Idealist"]
tags:
- Lab
- OS
- MIT 6.S081
mermaid: false #是否开启mermaid
# summary->在列表页展现的摘要内容，自动生成，内容默认前70个字符，可通过此参数自定义，一般无需专门设置
summary: ""
# description->需要自己编写的文章描述，是搜索引擎呈现在搜索结果链接下方的网页简介，建议设置
description: "MIT 6.S081 实验二 笔记"
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
    image: "https://cdn.jsdelivr.net/gh/Idealist226/img/imagelinux-computer.webp"
    caption: ""
    alt: ""
    relative: false
---

## 0. 引言

完成实验一后对本课程的编码模式有了一定的了解，接下来将继续完成实验二。

> 课程地址：<https://pdos.csail.mit.edu/6.S081/2020/schedule.html>
>
> 课程视频翻译：<https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/>
>
> 我的代码地址：<https://github.com/Idealist226/xv6-labs-2020>

## 1. 准备

```shell
git fetch
git checkout syscall
git branch -b syscall_test
make clean
```

## 2. Lab 2: system calls

> 实验地址：<https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html>

In the last lab you used systems calls to write a few utilities. In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.

在上一个实验中，您使用系统调用编写了一些实用程序。 在本实验中，您将向 xv6 添加一些新的系统调用，这将帮助您了解它们的工作原理，并让您了解 xv6 内核的一些内部结构。 您将在以后的实验中添加更多系统调用。

### System call tracing (moderate)

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

根据实验的 HINTS，一步步进行。

- HINT1：在 Makefile 的 UPROGS 中添加 `$U/_trace`。

- HINT2：

  1. 在 `user/user.h` 中添加系统调用的原型。

     ```c
     // user/trace.c
     if (trace(atoi(argv[1])) < 0) {
         fprintf(2, "%s: trace failed\n", argv[0]);
         exit(1);
     }
     ```

     在执行完 `make qemu` 后，发现 `user/trace.c` 未能编译，因为系统调用的用户空间存根尚不存在。

     观察 `user/trace.c` 中调用的 `trace()` 方法，其接受一个 int 型掩码，返回 int，并且如果执行失败返回负值。因此，在 `user/user.h` 中添加：

     ```c
     int trace(int);
     ```

  2. 在 `user/usys.pl` 中添加一个占位。

     Makefile 调用 perl 脚本 `user/usys.pl` 来生成 `user/usys.S`，`user/usys.S`也就是实际的系统调用存根（stub），它使用 RISC-V 的 `ecall` 指令来进入内核态。

     ``` perl
     # user/usys.pl
     sub entry {
         my $name = shift;
         print ".global $name\n";
         print "${name}:\n";
         print " li a7, SYS_${name}\n";
         print " ecall\n";
         print " ret\n";
     }
     ```

     - li：是一个加载立即数到寄存器的指令，格式是 `li rd, immediate`，表示将立即数 immediate 加载到寄存器 rd 中。
     - SYS_${name}：`kernel/syscall.h` 中 `#define SYS_${name} xxx` 的系统调用号，这样就可以告诉 ecall 要调用几号系统调用。（可见本 HINT 下方的第 3 步）

     - ecall：系统调用的 ecall 指令会使用 a0 和 a7 寄存器，其中 a7 寄存器保存的是系统调用号，a0 寄存器保存的是系统调用参数，返回值会保存在 a0 寄存器中。

     因此添加如下代码：

     ```perl
     entry("trace");
     ```

  3. 在 `kernel/syscall.h` 中添加一个系统调用号。

     ```c
     #define SYS_trace  22
     ```

- HINT3：在 `kernel/sysproc.c` 中添加一个 `sys_trace()` 函数，该函数通过在 `proc` 结构中的新变量中记住它的参数来实现新的系统调用（参见 `kernel/proc.h`）。 从用户空间检索系统调用参数的函数在 `kernel/syscall.c` 中，您可以在 `kernel/sysproc.c` 中看到它们的使用示例。

  1. 查看 `kernel/syscall.c` 中的函数。

     ```c
     // Fetch the nth 32-bit system call argument.
     int argint(int n, int *ip) {
       	*ip = argraw(n);
       	return 0;
     }
     
     static uint64 argraw(int n) {
       	struct proc *p = myproc();
       	switch (n) {
       	case 0:
         	return p->trapframe->a0;
       	case 1:
         	return p->trapframe->a1;
       	case 2:
         	return p->trapframe->a2;
       	case 3:
         	return p->trapframe->a3;
       	case 4:
         	return p->trapframe->a4;
       	case 5:
         	return p->trapframe->a5;
       	}
       	panic("argraw");
       	return -1;
     }
     ```

     由于内核与用户进程的页表不同，寄存器也不互通，所以参数无法直接通过 C 语言参数的形式传递过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe（用户进程陷入内核之前的寄存器等上下文信息）中读取用户进程寄存器中的参数。

  2. 在 `kernel/proc.h` 的 `proc` 结构体中添加一个新变量用于存储掩码。

     ```c
     struct proc {
       	// ...
       	char name[16];               // Process name (debugging)
       	int trace_mask;              // trace mask for debugging
     };
     ```

  3. 在 `kernel/sysproc.c` 中添加 `sys_trace()` 函数。

     该文件的其他系统调用函数的参数都是 void，因为获取参数使用的是 `argint()` 方法。

     结合这个文件里其他函数我们能看出 `myproc()` 函数可以获取当前进程的 PCB。

     ```c
     uint64 sys_trace(void) {
         int mask;
     
         if (argint(0, &mask) < 0) {
             return -1;
         }
         myproc()->trace_mask = mask;
         return 0;
     }
     ```

- HINT4：修改 `kernel/proc.c` 中的 `fork()` 以将掩码从父进程复制到子进程。

  ```c
  np->trace_mask = p->trace_mask;		// NEW
  ```

- HINT5：修改 `kernel/syscall.c` 中的 `syscall()` 方法来打印 trace 的输出信息。你需要添加一组系统调用名称以对其进行索引。

  1. 打印格式

     >  3: syscall read -> 1023

     打印出的一行包括：进程 id，系统调用的名字以及系统调用的返回值。

  2. 用 `extern` 全局声明新的内核调用函数，并且在 `syscalls` 映射表中，加入从前面定义的编号到系统调用函数指针的映射。

     ```c
     extern uint64 sys_trace(void);	// NEW
     
     static uint64 (*syscalls[])(void) = {
     // ...
       	[SYS_close]   sys_close,
       	[SYS_trace]   sys_trace,	// NEW
     };
     ```

     其中，syscalls 是一个指向函数指针的静态数组，该数组的参数为 void，返回值为 uint64。该数组中对于每一个系统调用 name，在数组的 `SYS_name` 下标中存放了`sys_name` 函数的地址。

     这种定义形式的理解可见 **[此处](https://stackoverflow.com/questions/26023270/c-function-explanation)**。

  3. 添加一组系统调用名称。

     ```c
     static char* syscall_names[] = {
         [SYS_fork]    "fork",
         [SYS_exit]    "exit",
         [SYS_wait]    "wait",
         [SYS_pipe]    "pipe",
         [SYS_read]    "read",
         [SYS_kill]    "kill",
         [SYS_exec]    "exec",
         [SYS_fstat]   "fstat",
         [SYS_chdir]   "chdir",
         [SYS_dup]     "dup",
         [SYS_getpid]  "getpid",
         [SYS_sbrk]    "sbrk",
         [SYS_sleep]   "sleep",
         [SYS_uptime]  "uptime",
         [SYS_open]    "open",
         [SYS_write]   "write",
         [SYS_mknod]   "mknod",
         [SYS_unlink]  "unlink",
         [SYS_link]    "link",
         [SYS_mkdir]   "mkdir",
         [SYS_close]   "close",
         [SYS_trace]   "trace",
     };
     ```

  4. 在 `syscall()` 方法中实现打印操作。

     ```c
     void syscall(void) {
         int num;
         struct proc *p = myproc();
         num = p->trapframe->a7;
         if (num > 0 && num < NELEM(syscalls) && syscalls[num]) { // 如果系统调用编号有效
             p->trapframe->a0 = syscalls[num]();	// 通过系统调用编号，获取系统调用处理函数的指针，调用并将返回值存到用户进程的 a0 寄存器中
             /* 如果当前进程设置了对该编号系统调用的 trace，则打出 pid、系统调用名称和返回值。*/
             if ((p->trace_mask >> num) & 1) {	// NEW
                 printf("%d: syscall %s -> %dn", p->pid, syscall_names[num], p->trapframe->a0);	  // NEW
             }									// NEW
         }
         else {
             printf("%d %s: unknown sys call %dn",
                    p->pid, p->name, num);
             p->trapframe->a0 = -1;
         }
     }   
     ```


整体的系统调用流程如下图所示：

![trace](https://cdn.jsdelivr.net/gh/Idealist226/img/imagetrace.png)

### Sysinfo (moderate)

> In this assignment you will add a system call, `sysinfo`, that collects information about the running system. The system call takes one argument: a pointer to a `struct sysinfo` (see `kernel/sysinfo.h`). The kernel should fill out the fields of this struct: the `freemem` field should be set to the number of bytes of free memory, and the `nproc` field should be set to the number of processes whose `state` is not `UNUSED`. We provide a test program `sysinfotest`; you pass this assignment if it prints "sysinfotest: OK".

- HINT1：在 Makefile 的 UPROGS 中添加 `$U/_sysinfotest`。

- HINT2：以与之前的任务相同的步骤进行声明。

  - user/user.h

    ```c
    struct sysinfo;
    
    // system calls
    // ...
    int sysinfo(struct sysinfo *);
    ```

  - user/usys.pl

    ```perl
    entry("sysinfo");
    ```

  - kernel/syscall.h

    ```c
    #define SYS_sysinfo 23
    ```

  - kernel/syscall.c

    ```c
    extern uint64 sys_sysinfo(void);
    
    static uint64 (*syscalls[])(void) = {
    // ...
    [SYS_sysinfo] sys_sysinfo,
    };
    
    static char* syscall_names[] = {
    // ...
    [SYS_sysinfo] "sysinfo",
    };
    ```

  - kernel/sysproc.c

    ```c
    uint64 sys_sysinfo(void) {
     	return 0;
    }
    ```

- HINT3：sysinfo 需要复制一个 `struct sysinfo` 到用户空间。根据提示，阅读 `sys_fstat()` 函数和 `filestat()` 函数。

  ```c
  // **kernel/sysfile.c**
  uint64
  sys_fstat(void)
  {
    struct file *f;
    uint64 st; // user pointer to struct stat
  
    if(argfd(0, 0, &f) < 0 || argaddr(1, &st) < 0)
      return -1;
    return filestat(f, st);
  }
  ```

  ```c
  // **kernel/file.c**
  // Get metadata about file f.
  // addr is a user virtual address, pointing to a struct stat.
  int
  filestat(struct file *f, uint64 addr)
  {
    struct proc *p = myproc();
    struct stat st;
    
    if(f->type == FD_INODE || f->type == FD_DEVICE){
      ilock(f->ip);
      stati(f->ip, &st);
      iunlock(f->ip);
      if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
        return -1;
      return 0;
    }
    return -1;
  }
  ```

  这里 copyout 可以将内核的数据复制到用户进程中，其原型是 `int
  copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)`，将 len 个字节从 src 复制到给定页表中的虚拟地址 dstva，成功返回 0，错误返回 -1。

- HINT4：要获得空闲内存的数目，请向`kernel/kalloc.c`添加一个函数。

  阅读该文件中的 `kfree()` 和 `kalloc()` 后，可知内存被空闲链表 `freelist` 管理，且 `freelist` 是链表的头结点。

  因此想获得空闲内存的大小只需要遍历一下 `freelist` 即可。

  ```c
  uint64 count_free_mem(void) {
      struct run *r;
      uint64 count = 0;
  
      acquire(&kmem.lock);
      r = kmem.freelist;
      while (r) {
          count++;
          r = r->next;
      }
      release(&kmem.lock);
      return count * PGSIZE;
  }
  ```

- HINT5：要获得进程的数量，请在`kernel/proc.c`中添加一个函数。

  阅读本文件的 `allocproc()` 函数后可看出，这里的进程通过由 `struct proc` 构成的数组 `proc` 进行管理。

  因此我们仿照 `allocproc()` 来实现统计进程数量的函数。

  *注意：在这里，不需要锁进程  proc 结构，因为我们只需要读取进程列表，不需要写*。

  ```c
  uint64 count_process(void) {
      struct proc *p;
      uint64 count = 0;
  
      for (p = proc; p < &proc[NPROC]; p++) {
          if (p->state != UNUSED) {
              count++;
          }
      }
      return count;
  }
  ```

- 在 `kernel/defs.h` 中添加函数的原型。

  ```c
  // kalloc.c
  // ...
  uint64          count_free_mem(void);		// NEW
  
  // proc.c
  // ...
  uint64          count_process(void);		// NEW
  ```

- 在 `kernel/sysproc.c` 中实现 `sys_info` 函数。

  ```c
  uint64 sys_sysinfo(void) {
      struct sysinfo info;
      uint64 addr;
  
      info.freemem = count_free_mem();
      info.nproc = count_process();
      if (argaddr(0, &addr) < 0) {
          return -1;
      }
      if (copyout(myproc()->pagetable, addr, (char *)&info, sizeof(info)) < 0) {
          return -1;
      } else {
          return 0;
      }
  }
  ```

## 3. 总结

在本次 lab 中添加了一些新的系统调用，略微窥探了 xv6 的内部，有种揭开了它一角面纱的惊喜感。该 lab 下的两组任务给出了比较详细的提示，随着 HINTS 一步步做下来还是很顺畅的。期待后续的内容。

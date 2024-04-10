# xv6 book 读书笔记
## 1 Operating system interfaces
### 1.1 系统调用
。。。后续会讲
### 1.2 文件描述符、管道
A process may obtain a file descriptor by opening a file, directory, or device, or by creating a pipe, or by duplicating an existing descriptor.             
the xv6 kernel uses the file descriptor as an index into a per-process table, so that every process has a private space of file descriptors starting at zero.
#### 1.2.1 I/O重定向
Here is a simplified version of the code a shell runs for the command `cat < input.txt`.
```c
char *argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
    close(0);
    open("input.txt", O_RDONLY);
    exec("cat", argv);
}
```
#### 1.2.2 管道
* 使用pipe()创建管道和一次重定向实现父子进程之间的数据传递
```c
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if(fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
} else {
    close(p[0]);
    write(p[1], "hello world\n", 12);
    close(p[1]);
}
```
* 使用pipe()创建管道和两次重定向实现unix管道命令`grep fork sh.c | wc -l`
```c
case PIPE:
    pcmd = (struct pipecmd*)cmd;
    if(pipe(p) < 0)
      panic("pipe");
    if(fork1() == 0){
      close(1);
      dup(p[1]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->left);
    }
    if(fork1() == 0){
      close(0);
      dup(p[0]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->right);
    }
    close(p[0]);
    close(p[1]);
    wait(0);
    wait(0);
    break;
```
The `dup()` system call creates a copy of a file descriptor.

-   It uses the lowest-numbered unused descriptor for the new descriptor.
-   If the copy is successfully created, then the original and copy file descriptors may be used interchangeably.
-   They both refer to the same open file description and thus share file offset and file status flags.

这里使用了两次重定向，`fork()`了两次，但由于执行了`runcmd()`，因此只有两个子进程，一个子进程重定向了标准输出后执行`|`左边的命令，一个子进程重定向了标准输出后执行`|`右边的命令。
* pipe()系统调用创建了管道，管道是内核的缓冲区，自带进程间的同步效果。
##### 1.2.2.1 pipe()创建管道内核具体实现
。。。未完待续
### 1.3 文件系统
。。。后续会讲
## 2 Operating system organization
### 2.1 Abstracting physical resources
前面一章的接口给物理资源提供了一层抽象，操作系统使得进程可以分时复用硬件、共享硬件，并提供进程之间的隔离性与交互性。
### 2.2 User mode, supervisor mode, and system calls
CPUs provide hardware support for strong isolation. For example, RISC-V has three modes in which the CPU can execute instructions: machine mode, supervisor mode, and user mode.       
CPUs provide a special instruction that switches the CPU from user mode to supervisor mode and enters the kernel at an entry point specified by the kernel. (RISC-V provides the ecall instruction for this purpose.)
### 2.3 Kernel organization
xv6是*monolithic kernel*（宏内核），与之对应的是*microkernel*（微内核）。是宏微内核取决于内核代码是否全部在内核态下运行，是则是宏内核，不是则是微内核。
下面是xv6的内核代码构造：
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d857da415744e4a9fbbf7327e210eec~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=577&h=729&s=134159&e=png&b=fffefe)
### 2.4 Process overview
在虚拟内存系统中每个进程都有自己独立的从0开始的地址空间。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d21e34669b3f440398a472705cbd3edf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=490&h=374&s=19422&e=png&b=ffffff)

每个进程两个栈：用户栈和内核栈，这么设计的原因是：The kernel stack is separate (and protected from user code) so that the kernel can execute even if a process has wrecked its user stack.
### 2.5 Code: starting xv6 and the first process
下面是xv6内核启动的过程：
* RISC-V机器加电，初始化自己，运行存储在ROM中的boot loader，boot loader将xv6内核加载到内存，起始地址是0x80000000。
* 开始执行xv6 kernel/entry.S中的代码_entry函数，设置内核栈以便执行内核c函数。（_entry中的代码是要设置好内核栈stack0，以便下面执行xv6的c函数，并调用start函数）
* 执行xv6 kernel/start.c中的start()函数，在machine mode下设置mret用的一些东西。（start()函数在machine mode下设置了mstatus寄存器以备最后执行的mret返回至supervisor mode，设置了mepc寄存器以备mret返回至main()函数，设置satp寄存器为0以禁用页表，并对时钟芯片编程以产生定时器中断，最后执行内嵌汇编指令mret跳转到main()函数）
* mret之后，在supervisor mode下执行main()函数，main()函数中只有一个cpu会初始化一些内核服务和子系统（比如创建内核页表、设置页表基址、初始化buffer cache、inode cache、文件表等），并且调用userinit()函数初始化创建第一个用户进程。
* userinit()执行initcode.S中的汇编指令，调用exec()函数调用。
* exec()将第一个进程的地址空间替换为init.c对应可执行文件的。
* init在exec()sh.c对应的可执行文件。         
这一切搞完之后，第四步的main()函数每个cpu都会最后调用scheduler()函数运行每个cpu的调度线程进行进程的调度，第一个被调度的是刚刚被设置好的shell进程。

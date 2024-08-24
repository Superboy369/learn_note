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
[见第八章File system 8.6.2pipe(p)管道内核实现](#8.6.2-pipe(p)管道内核实现)

<a href="#8.6.2 pipe(p)管道内核实现">跳转到更新</a>

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
* mret之后，在supervisor mode下执行main()函数，main()函数中只有一个cpu会初始化一些内核服务和子系统（比如编程PLIC硬件写寄存器使能产生PLIC中断、创建内核页表、设置页表基址、初始化buffer cache、inode cache、文件表等），并且调用userinit()函数初始化创建第一个用户进程，其他的cpu只需要打开页表、编程当前cpu硬件写寄存器使能接收PLIC中断。
* userinit()执行initcode.S中的汇编指令，调用exec()函数调用。
* exec()将第一个进程的地址空间替换为init.c对应可执行文件的。
* init再在fork()之后子进程中exec()sh.c对应的可执行文件。         
这一切搞完之后，第四步的main()函数每个cpu都会最后调用scheduler()函数运行每个cpu的调度线程进行进程的调度，第一个被调度的是刚刚被设置好的shell进程。
## 3 Page tables
### 3.1 Paging hardware

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5df0b26ed170435695cbb393702a8c58~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=797&h=802&s=87587&e=png&b=fdfcfc)
* xv6虚拟地址位数占64/39bit(VA)(39 = 9 + 9 + 9 + 12)，物理地址位数占56bit(PA)(56 = 44 + 12)，每页$2^{12} = 4094$byte，每个PTE占64/54bit(54 = 44 + 10)。            
* 每个cpu一个satp寄存器。
* 指令使用的都是虚拟地址，硬件负责将PA转化为VA，然后将PA传给DRAM硬件来读写内存。
### 3.2 Kernel address space
xv6每个进程一个自己的用户页表和一个共用的内核页表，这个共用的内核页表的映射如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3372b932b2604b7ab97e6ae945584315~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=825&h=823&s=86544&e=png&b=fefefe)
内核地址空间都是直接映射，但是除了trampoline页和每个进程对应的内核栈被映射了两次，一次直接映射，一次虚拟高地址映射。
## 4 Traps and system calls
### 4.1 内核态和用户态之间的切换

*   用户态->内核态：内中断时用户态下执行特殊指令ecall，切换CPU位状态用户态为内核态，外中断时信号线改变cpu位状态。
*   内核态->用户态：内中断返回时内核态下执行和ecall相反的特殊指令sret，切换CPU位状态内核态为用户态。
*   ecall和sret指令的四件事情：
    *   关中断 */* 开中断
    *   用户态->内核态 */* 内核态->用户态
    *   pc->sepc、stvec->pc */* sepc->pc
    *   跳转至pc所指地址继续执行汇编代码

## 5 Interrupts and device drivers
### 5.1 操作系统控制设备的方式
操作系统以memory mapped的方式将设备控制寄存器mapped到物理内存，使得可以使用汇编指令来进行设备的控制。
```c
// the UART control registers are memory-mapped
// at address UART0. this macro returns the
// address of one of the registers.
#define Reg(reg) ((volatile unsigned char *)(UART0 + reg))

// the UART control registers.
// some have different meanings for
// read vs write.
// see http://byterunner.com/16550.html
#define RHR 0 // receive holding register (for input bytes)
#define THR 0 // transmit holding register (for output bytes)
#define IER 1 // interrupt enable register
#define IER_RX_ENABLE (1<<0)
#define IER_TX_ENABLE (1<<1)
#define FCR 2 // FIFO control register
#define FCR_FIFO_ENABLE (1<<0)
#define FCR_FIFO_CLEAR (3<<1) // clear the content of the two FIFOs
#define ISR 2 // interrupt status register
#define LCR 3 // line control register
#define LCR_EIGHT_BITS (3<<0)
#define LCR_BAUD_LATCH (1<<7) // special mode to set baud rate
#define LSR 5 // line status register
#define LSR_RX_READY (1<<0) // input is waiting to be read from RHR
#define LSR_TX_IDLE (1<<5) // THR can accept another character to send
  
#define ReadReg(reg) (*(Reg(reg)))
#define WriteReg(reg, v) (*(Reg(reg)) = (v))
```
不仅仅是设备，cpu本身也会有控制寄存器映射到物理内存中。如PLIC和cpu中都有控制寄存器来使能PLIC中断，一个是使能产生PLIC中断，一个是使能接收PLIC中断。
```c
// qemu puts platform-level interrupt controller (PLIC) here.
#define PLIC 0x0c000000L
#define PLIC_PRIORITY (PLIC + 0x0)
#define PLIC_PENDING (PLIC + 0x1000)
#define PLIC_MENABLE(hart) (PLIC + 0x2000 + (hart)*0x100)
#define PLIC_SENABLE(hart) (PLIC + 0x2080 + (hart)*0x100)
#define PLIC_MPRIORITY(hart) (PLIC + 0x200000 + (hart)*0x2000)
#define PLIC_SPRIORITY(hart) (PLIC + 0x201000 + (hart)*0x2000)
#define PLIC_MCLAIM(hart) (PLIC + 0x200004 + (hart)*0x2000)
#define PLIC_SCLAIM(hart) (PLIC + 0x201004 + (hart)*0x2000)
```
### 5.2 内核main()对设备的初始化
每个cpu都会执行`plicinithart()`来使能当前cpu接收PLIC中断。只有一个cpu会执行`consoleinit()`来通过写uart寄存器的方式设置uart设备，执行`plicinit()`来使能PLIC产生中断。
```c
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
    if(cpuid() == 0){
        consoleinit();
        printfinit();
        printf("\n");
        printf("xv6 kernel is booting\n");
        printf("\n");
        kinit(); // physical page allocator
        kvminit(); // create kernel page table
        kvminithart(); // turn on paging
        procinit(); // process table
        trapinit(); // trap vectors
        trapinithart(); // install kernel trap vector
        plicinit(); // set up interrupt controller
        plicinithart(); // ask PLIC for device interrupts
        binit(); // buffer cache
        iinit(); // inode cache
        fileinit(); // file table
        virtio_disk_init(); // emulated hard disk
        userinit(); // first user process
        __sync_synchronize();
        started = 1;
    } else {
        while(started == 0)
            ;
        __sync_synchronize();
        printf("hart %d starting\n", cpuid());
        kvminithart(); // turn on paging
        trapinithart(); // install kernel trap vector
        plicinithart(); // ask PLIC for device interrupts
    }
    scheduler();
}
```
### 5.3 device driver
设备驱动程序分为两部分：
* top：负责系统调用接口，如read()、write()，数据从用户区->设备
* bottom：负责设备对应的中断处理程序，数据从设备->用户区

以uart.c驱动程序为例，top部分有函数`uartputc()`向uart_tx_buffer中添加字符，`uartgetc()`从uart_tx_buffer中获取字符。
```c
// add a character to the output bufyoufer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
    acquire(&uart_tx_lock);
    if(panicked){
        for(;;)
            ;
    }
    while(1){
        if(((uart_tx_w + 1) % UART_TX_BUF_SIZE) == uart_tx_r){
            // buffer is full.
            // wait for uartstart() to open up space in the buffer.
            sleep(&uart_tx_r, &uart_tx_lock);
        } else {
            uart_tx_buf[uart_tx_w] = c;
            uart_tx_w = (uart_tx_w + 1) % UART_TX_BUF_SIZE;
            uartstart();
            release(&uart_tx_lock);
            return;
        }
    }
}

// read one input character from the UART.
// return -1 if none is waiting.
int
uartgetc(void)
{
    if(ReadReg(LSR) & 0x01){
        // input data is ready.
        return ReadReg(RHR);
    } else {
        return -1;
    }
}
```
uart.c驱动程序bottom部分有函数`uartintr()`uart设备的中断处理函数。

而console.c驱动程序top部分有函数`consputc()`被`printf()`所调用且调用`uartputc()`向uart的传输寄存器写字符，函数`consolewrite()`被`write(console)`所调用且调用`uartputc()`向uart_tx_buffer写字符，函数`consoleread()`被`read(console)`所调用且从uart_tx_buffer中获取字符。
### 5.4 I/O方式
I/O操作是指将进行I/O设备和内存的数据交互。有三种数据交互方式：
1. polling（轮询）：cpu和I/O设备并发运行，程序轮询I/O寄存器，查看数据是否准备好。
2. 中断：cpu和I/O设备并行运行，I/O设备准备好一字数据后，以中断的方式通知cpu拿走数据。
3. DMA：I/O设备使用DMA控制器来控制设备到内存的数据传输，一批数据都准备好后才中断cpu。

- 轮询方式cpu会自旋等待浪费cpu资源，但是省去了中断切换操作系统状态的开销。中断方式则正好相反，不会浪费cpu资源，但是需要中断切换操作系统状态的开销（开销并不小，需要很多个指令周期）。
- 对于频繁申请I/O操作的高速设备，如网卡，可以使用轮询的I/O方式。对于低速设备，如键盘、鼠标，可以使用中断的I/O方式。
### 5.5 shell打印'$'（字符从程序用户区->console设备）（write(硬件设备)系统调用）的背后逻辑
`sh.c/fprintf(2, "$ ")`->`printf.c/vprintf(fd, fmt, ap)`->`printf.c/putc(fd, c)`->system call`write()`->`usys.pl/ecall`->`trampoline.S`->`trap.c/usertrap()`->`syscall()`->`sysfile.c/sys_write()`->`file.c/filewrite()`->`console.c/consolewrite()`->`uart.c/uartputc()`->将第一个字符'\$'放在uart_tx_buffer中->`uart.c/uartstart()`->`WriteReg(THR,c)`->返回，之后uart硬件将'$'发送到console设备上，console画出来。

每次uart第一个字符发送完，uart都会产生中断->经过和system call一样的trap机制`trap.c/usertrap()`->`trap.c/devintr()`->`uart.c/uartintr()`->`uartgetc()`、`consoleintr(c)`、`uartstart()`->`WriteReg(THR,c)`->返回，之后uart硬件将后续字符发送到console设备上，console画出来。
### 5.6 键盘敲字到console和用户区中（字符从uart设备->程序用户区/console设备）（设备中断）的背后逻辑
type 'ls'到uart硬件中->uart产生中断->经过和system call一样的trap机制`trap.c/usertrap()`->`trap.c/devintr()`->`uart.c/uartintr()`->`uart.c/uartgetc()`从uart硬件中读一个字符、`console.c/consoleintr()`将字符累计一行在cons.buf中->`console.c/consoleread()`将cons.buf中的字符copy到用户区->之后返回至用户区的中断处继续执行。
## 6 Locking
### 6.1 锁

锁在xv6操作系统中就是一个结构体，里面包含了locked字段和一些用于debug的字段（name、持有锁的cpu）。

*   acquire(\&lock)会先关中断，再利用硬件原子执行test\_and\_set locked字段。
*   release(\&lock)会先利用硬件原子执行test\_and\_set locked字段，再开中断。
*   开关中断意味着在临界区代码中无法中断，因为在临界区中断可能导致死锁（普通程序在acquire()之后被中断，之后中断处理程序对同一把锁上锁导致死锁）。

## 7 Scheduling
### 7.1 进程切换过程
cpu的执行流不断在不同进程和cpu调度器代码中来回切换。进程要么主动调用sleep()释放cpu，sleep()最终调用swtch()切换到cpu调度器代码中的swtch()，要么当进程被时钟中断后被动调用yield()释放cpu，yield()最终调用swtch()切换到cpu调度器代码中的swtch()（切换当前进程上下文为cpu调度器上下文）。cpu调度器则会从swtch()继续执行，寻找下一个RUNNABLE的进程，并调用swtch()切换到被调度进程代码中的swtch()（切换cpu调度器上下文为被调度进程上下文），这样就完成了进程的调度和进程上下文的保存切换。
```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
    int which_dev = 0;
    if((r_sstatus() & SSTATUS_SPP) != 0)
        panic("usertrap: not from user mode");
    // send interrupts and exceptions to kerneltrap(),
    // since we're now in the kernel.
    w_stvec((uint64)kernelvec);
    struct proc *p = myproc();
    // save user program counter.
    p->trapframe->epc = r_sepc();
    if(r_scause() == 8){
        // system call
        if(p->killed)
            exit(-1);
        // sepc points to the ecall instruction,
        // but we want to return to the next instruction.
        p->trapframe->epc += 4;
        // an interrupt will change sstatus &c registers,
        // so don't enable until done with those registers.
        intr_on();
        syscall();
    } else if((which_dev = devintr()) != 0){
        // ok
    } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        printf(" sepc=%p stval=%p\n", r_sepc(), r_stval());
        p->killed = 1;
    }
    if(p->killed)
        exit(-1);
    // give up the CPU if this is a timer interrupt.
    if(which_dev == 2)
        yield();  // 当前进程在时钟中断时内核调用yield()被动释放cpu
    usertrapret();
}
```
```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
    struct proc *p = myproc();
    acquire(&p->lock);  
    p->state = RUNNABLE;
    sched();  // 当前进程被动调用sched()
    release(&p->lock);  
}
```
```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
    struct proc *p = myproc();
    // Must acquire p->lock in order to
    // change p->state and then call sched.
    // Once we hold p->lock, we can be
    // guaranteed that we won't miss any wakeup
    // (wakeup locks p->lock),
    // so it's okay to release lk.
    if(lk != &p->lock){ //DOC: sleeplock0
        acquire(&p->lock); //DOC: sleeplock1  
        release(lk); 
    }
    // Go to sleep.
    p->chan = chan;
    p->state = SLEEPING;
    sched();  // 当前进程主动调用sleep()放弃cpu，主动调用sched()
    // Tidy up.
    p->chan = 0;
    // Reacquire original lock.
    if(lk != &p->lock){
        release(&p->lock);  
        acquire(lk);  
    }
}
```
```c
// Switch to scheduler. Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
    int intena;
    struct proc *p = myproc();
    if(!holding(&p->lock))
        panic("sched p->lock");
    if(mycpu()->noff != 1)
    panic("sched locks");
    if(p->state == RUNNING)
        panic("sched running");
    if(intr_get())
        panic("sched interruptible");
    intena = mycpu()->intena;
    swtch(&p->context, &mycpu()->context);  // sched()最终调用swtch()保存当前进程上下文，回复cpu调度器上下文 
    mycpu()->intena = intena;
}
```
```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns. It loops, doing:
// - choose a process to run.
// - swtch to start running that process.
// - eventually that process transfers control
// via swtch back to the scheduler.
void
scheduler(void)
{
    struct proc *p;
    struct cpu *c = mycpu();
    c->proc = 0;
    for(;;){
        // Avoid deadlock by ensuring that devices can interrupt.
        intr_on();
        int nproc = 0;
            for(p = proc; p < &proc[NPROC]; p++) {
                acquire(&p->lock); 
                if(p->state != UNUSED) {
                    nproc++;
                }
                if(p->state == RUNNABLE) {
                    // Switch to chosen process. It is the process's job
                    // to release its lock and then reacquire it
                    // before jumping back to us.
                    p->state = RUNNING;
                    c->proc = p;
                    swtch(&c->context, &p->context);  // 每个cpu调度器scheduler()调用swtch()保存cpu调度器的上下文，回复被调度进程的上下文
                    // Process is done running for now.
                    // It should have changed its p->state before coming back.
                    c->proc = 0;
                }
                release(&p->lock); 
            }
            if(nproc <= 2) { // only init and sh exist
            intr_on();
            asm volatile("wfi");
        }
    }
}
```
### 7.2 fork()中父子进程的行为

![fork()系统调用父子进程行为](https://github.com/user-attachments/assets/b8154d5b-85a4-4d17-945c-2c4fd4333fed)

在fork()调用allocproc()为子进程分配pcb（在xv6中是proc结构体）的时候中会将context.ra设置为forkret()函数的地址，因此fork()后的子进程被调度之后会首先跳转到forkret()中，这样做是因为子进程和父进程被切换调度时的断点是不一样的，他们是两个独立调度的进程。
```c
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}
```

### 7.3 进程切换中对于进程锁和非进程锁的行为

*   在进程切换前要持有当前进程的锁并且释放出当前进程锁的其他锁之后在切换至其他进程。
*   在进程切换之前/后（yield()和sleep()调用sched()(swtch())之前/后）要持有/释放当前要下处理机的进程的锁的原因是：如果不持有当前进程锁，多个cpu scheduler()在调度进程时都调度同一个进程上处理机。
*   在进程切换之前/后（yield()和sleep()调用sched()/swtch()之前/后）要释放/持有除当前要下处理机的进程的锁之外的其他所有设备锁的原因是：如果不释放，会导致死锁，当调度到的其他进程在对该设备上锁时会关中断并进入自旋等待，而持有该设备锁的进程由于该cpu关了中断而无法上处理机释放锁，导致死锁。

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns. It loops, doing:
// - choose a process to run.
// - swtch to start running that process.
// - eventually that process transfers control
// via swtch back to the scheduler.
void
scheduler(void)
{
    struct proc *p;
    struct cpu *c = mycpu();
    c->proc = 0;
    for(;;){
        // Avoid deadlock by ensuring that devices can interrupt.
        intr_on();
        int nproc = 0;
            for(p = proc; p < &proc[NPROC]; p++) {
                acquire(&p->lock);  // @@@获取当前进程锁
                if(p->state != UNUSED) {
                    nproc++;
                }
                if(p->state == RUNNABLE) {
                    // Switch to chosen process. It is the process's job
                    // to release its lock and then reacquire it
                    // before jumping back to us.
                    p->state = RUNNING;
                    c->proc = p;
                    swtch(&c->context, &p->context);
                    // Process is done running for now.
                    // It should have changed its p->state before coming back.
                    c->proc = 0;
                }
                release(&p->lock);  // @@@释放当前进程锁
            }
            if(nproc <= 2) { // only init and sh exist
            intr_on();
            asm volatile("wfi");
        }
    }
}
```

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
    struct proc *p = myproc();
    acquire(&p->lock);  // @@@获取当前进程锁
    p->state = RUNNABLE;
    sched();
    release(&p->lock);  // @@@释放当前进程锁
}
```

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
    struct proc *p = myproc();
    // Must acquire p->lock in order to
    // change p->state and then call sched.
    // Once we hold p->lock, we can be
    // guaranteed that we won't miss any wakeup
    // (wakeup locks p->lock),
    // so it's okay to release lk.
    if(lk != &p->lock){ //DOC: sleeplock0
        acquire(&p->lock); //DOC: sleeplock1  // @@@获取当前进程锁
        release(lk);  // @@@@释放非进程锁
    }
    // Go to sleep.
    p->chan = chan;
    p->state = SLEEPING;
    sched();
    // Tidy up.
    p->chan = 0;
    // Reacquire original lock.
    if(lk != &p->lock){
        release(&p->lock);  // @@@释放当前进程锁
        acquire(lk);  // @@@@重新对非进程锁上锁
    }
}
```

```c
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
    struct proc *p;
    for(p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);  // @@@获取当前进程锁
        if(p->state == SLEEPING && p->chan == chan) {
            p->state = RUNNABLE;
        }
        release(&p->lock);  // @@@释放当前进程锁
    }
}
```

```c
// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
    // read and process incoming characters.
    while(1){
        int c = uartgetc();
        if(c == -1)
            break;
        consoleintr(c);
    }
    // send buffered characters.
    acquire(&uart_tx_lock);  // @@@@互斥访问uart
    uartstart();
    release(&uart_tx_lock);  // @@@@互斥访问uart
}
```

### 7.4 进程的退出和终止

xv6有两种停止进程的方式：1.exit()，2.kill()：

*   exit():exit()只是关文件，设置当前进程state为ZOMBIE，并调用sched()，真正资源的free在父进程被调度运行wait()时，子进程的资源由父进程来关闭。init()进程会作为所有无父进程进程的父进程调用wait()释放其子进程的资源。
*   kill():kill()系统调用只是设置p->killed字段为1，并设置进程state为RUNNABLE，进程被调度运行时用于中断陷入usertrap()中，在usertrap()中判断p->killed为1调用exit(-1)。
*   所有main()函数最后都会隐式的调用exit()函数来让父进程释放资源。
*   这些资源包括页表、trapframe、物理页、一系列proc结构体中的数据。

```c
// Exit the current process. Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
    struct proc *p = myproc();
    if(p == initproc)
        panic("init exiting");
    // Close all open files.
    for(int fd = 0; fd < NOFILE; fd++){
        if(p->ofile[fd]){
            struct file *f = p->ofile[fd];
            fileclose(f);
            p->ofile[fd] = 0;
        }
    }
    begin_op();
    iput(p->cwd);
    end_op();
    p->cwd = 0;
        // we might re-parent a child to init. we can't be precise about
    // waking up init, since we can't acquire its lock once we've
    // acquired any other proc lock. so wake up init whether that's
    // necessary or not. init may miss this wakeup, but that seems
    // harmless.
    acquire(&initproc->lock);
    wakeup1(initproc);
    release(&initproc->lock);
    // grab a copy of p->parent, to ensure that we unlock the same
    // parent we locked. in case our parent gives us away to init while
    // we're waiting for the parent lock. we may then race with an
    // exiting parent, but the result will be a harmless spurious wakeup
    // to a dead or wrong process; proc structs are never re-allocated
    // as anything else.
    acquire(&p->lock);
    struct proc *original_parent = p->parent;
    release(&p->lock);
    // we need the parent's lock in order to wake it up from wait().
    // the parent-then-child rule says we have to lock it first.
    acquire(&original_parent->lock);
    acquire(&p->lock);
    // Give any children to init.
    reparent(p);
    // Parent might be sleeping in wait().
    wakeup1(original_parent);
    p->xstate = status;
    p->state = ZOMBIE;
    release(&original_parent->lock);
    // Jump into the scheduler, never to return.
    sched();
    panic("zombie exit");
}
```

```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
    struct proc *np;
    int havekids, pid;
    struct proc *p = myproc();
    // hold p->lock for the whole time to avoid lost
    // wakeups from a child's exit().
    acquire(&p->lock);
    for(;;){
        // Scan through table looking for exited children.
        havekids = 0;
        for(np = proc; np < &proc[NPROC]; np++){
            // this code uses np->parent without holding np->lock.
            // acquiring the lock first would cause a deadlock,
            // since np might be an ancestor, and we already hold p->lock.
            if(np->parent == p){
                // np->parent can't change between the check and the acquire()
                // because only the parent changes it, and we're the parent.
                acquire(&np->lock);
                havekids = 1;
                if(np->state == ZOMBIE){
                    // Found one.
                    pid = np->pid;
                    if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,sizeof(np->xstate)) < 0) {
                        release(&np->lock);
                        release(&p->lock);
                        return -1;
                    }
                    freeproc(np);
                    release(&np->lock);
                    release(&p->lock);
                    return pid;
                }
                release(&np->lock);
            }
        }
        // No point waiting if we don't have any children.
        if(!havekids || p->killed){
            release(&p->lock);
            return -1;
        }
        // Wait for a child to exit.
        sleep(p, &p->lock); //DOC: wait-sleep
    }
}
```

```c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
    struct proc *p;
    for(p = proc; p < &proc[NPROC]; p++){
        acquire(&p->lock);
        if(p->pid == pid){
            p->killed = 1;
            if(p->state == SLEEPING){
                // Wake process from sleep().
                p->state = RUNNABLE;
            }
            release(&p->lock);
            return 0;
        }
        release(&p->lock);
    }
    return -1;
}
```

## 8 File system
### 8.1 xv6的文件系统
xv6的文件系统是存储在 qemu 模拟出来的虚拟磁盘中的。在调用linux命令`make qemu`后:
1. 自动编译链接内核中的所有.c文件
2. 创建并且使用编译好的`mkfs`可执行文件将这些文件都写入`fs.img`镜像文件中，并将其加载至 qemu 虚拟磁盘中。
#### 8.1.1 mkfs文件
在xv6中mkfs是个c文件，在make编译好之后被`make qemu`执行命令`mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)`运行可执行文件mkfs
* mkfs创建打开了`fs.img`文件
* mkfs将内核中的文件都读到了`fs.img`文件中
* mkfs初始化了超级块中的数据，主要是定义了文件系统的大小、磁盘块的数量、日志块的数量、日志块的开始编号、inode块的开始编号、bitmap块的开始编号。
### 8.2 buffer cache layer
buffer cache 作为外存（磁盘、固态硬盘）在内存中的缓存，其实现保证了：
* 多个进程对同一块 disk block 访问的互斥性。
* 根据程序的局部性原理缓存最近可能使用的 disk block ，减少系统对外存的访问，提高访问效率。
#### 8.2.1 bread()
bread() 从外存中读取 disk block 内容到 buffer cache 内存中。
```c
// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
    struct buf *b;
    b = bget(dev, blockno); // 获取返回的指定设备和 block 号的 buffer cache （没有的话会分配后返回）
    if(!b->valid) {
        virtio_disk_rw(b, 0); // 从外存中读内容到 buffer cache 中
        b->valid = 1;
    }
    return b;
}
```
* bread() 中调用 bget() ，bget()会返回上过锁的 buffer cache 块，bget() 返回的指定设备和 block 号的 buffer cache ，没有的话会按照分配后返回。
```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
    struct buf *b;
    acquire(&bcache.lock);
    // Is the block already cached?
    for(b = bcache.head.next; b != &bcache.head; b = b->next){
        if(b->dev == dev && b->blockno == blockno){
            b->refcnt++;
            release(&bcache.lock);
            acquiresleep(&b->lock);
            return b;
        }
    }
    // Not cached.
    // Recycle the least recently used (LRU) unused buffer.
    for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
        if(b->refcnt == 0) {
            b->dev = dev;
            b->blockno = blockno;
            b->valid = 0;
            b->refcnt = 1;
            release(&bcache.lock);
            acquiresleep(&b->lock);
            return b;
        }
    }
    panic("bget: no buffers");
}
```
#### 8.2.2 bwrite()
bwrite() 将 buffer cache 内存中的内容写到外存中。
```c
// Write b's contents to disk. Must be locked.
void
bwrite(struct buf *b)
{
    if(!holdingsleep(&b->lock))
    panic("bwrite");
    virtio_disk_rw(b, 1);
}
```
#### 8.2.3 buffer cache LRU 替换策略的实现
* 在 xv6 中使用双向链表实现的 buffer cache 的 LRU 策略，双向链表中头部代表最近刚刚使用完，尾部表示最近最少使用。
* 双向链表在 xv6 内核启动时初始化好。
  * 然后在 bread() 调用 bget() 时，如果已经 cache 过，就将 refcnt 加一后返回上锁之后的 buffer cache，如果没有 cache 过，就按照 LRU 策略从后往前扫描双向链表，获取第一个 refcnt == 0 的 buffer cache 上锁之后返回。
  * 在 caller 使用完 buffer cache 之后调用 brelse() 时，brelse() 会将 refcnt 减一之后，判断 refcnt 是否为0，如果不为0什么也不干直接释放锁，如果为0说明没有一个进程在继续使用这个 buffer cache 了，所以把它移至双向链表头部表示最近经常使用。
#### 8.2.4 buffer cache 中的锁
xv6 中的 buffer cache 涉及到两个锁：
* bcache->lock:保证多个进程对所有 buffer cache 所构成的双向链表的访问。
* buf 的 lock:保证多个进程对单个 buffer cache 的读写的原子性。
```c
struct {
    struct spinlock lock;
    struct buf buf[NBUF];
    // Linked list of all buffers, through prev/next.
    // Sorted by how recently the buffer was used.
    // head.next is most recent, head.prev is least.
    struct buf head;
} bcache;

struct buf {
    int valid; // has data been read from disk?
    int disk; // does disk "own" buf?
    uint dev;
    uint blockno;
    struct sleeplock lock;
    uint refcnt;
    struct buf *prev; // LRU cache list
    struct buf *next;
    uchar data[BSIZE];
};
```
### 8.3 logging layer
#### 8.3.1 crash 恢复

![xv6 crash恢复](https://github.com/user-attachments/assets/57d3f47a-5644-4d05-9efd-30f191c35c04)


#### 8.3.2 xv6文件系统调用log日志实现

![xv6文件系统调用日志具体实现](https://github.com/user-attachments/assets/f11f6c9e-7679-48f5-ae52-0df39382c468)
```c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}

// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}

static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}

// Copy modified blocks from cache to log.
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log
    brelse(from);
    brelse(to);
  }
}

// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}

// Copy committed blocks from log to their home location
static void
install_trans(int recovering)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    if(recovering == 0)
      bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}
```

#### 8.3.3 xv6具体crash恢复流程

![xv6_crash恢复流程](https://github.com/user-attachments/assets/5e871c0e-bf5a-4554-92de-6195fc0b6331)

xv6crash恢复是在forkret()中恢复的，而forkret()只有在fork()后的子进程第一次被cpu scheduler()调用的时候会调用。
```c
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```

#### 8.3.4 其他一些挑战
##### 8.3.4.1 涉及log block的cache驱逐
如果buffer cache要驱逐还未写入log block的block至实际的block中，这会出现问题，因为log设计需要buffer cache块中的block必须先写入log block，再写入实际block中。xv6是通过将`refcnt + 1`来解决这个问题的。在log_write()更新log结构体的时候调用bpin() refcnt + 1，当install_trans() install log之后调用bunpin() refcnt - 1。
```c
void
bpin(struct buf *b) {
  acquire(&bcache.lock);
  b->refcnt++;
  release(&bcache.lock);
}

void
bunpin(struct buf *b) {
  acquire(&bcache.lock);
  b->refcnt--;
  release(&bcache.lock);
}
```

##### 8.3.4.2 fs op must fit in log
log block数量必须大于fs一次性写入block的最大数量，因为一次性写入block数量太大会导致log block装不下一次写入的数据，导致破坏了log实现的不变性。当一次写入数量较大的block时（写很多数据到文件中），xv6会将一个写入事务拆分未多个较小的事务来进行写入操作。

##### 8.3.4.3 多个进程事务的并发
多个进程的fs system call可能会导致log block数量不够，xv6通过限制并发的fs system syscall的数量来避免。

### 8.4 inode layer
#### 8.4.1 inode layer和log layer下的系统调用流程

![inode layer和log layer的exec()系统调用流程](https://github.com/user-attachments/assets/8963ea38-525e-40a3-a868-51d43299363e)

![inode layer和log layer的read()系统调用流程](https://github.com/user-attachments/assets/816f9ec8-5ebb-4c19-9d11-ba106f260f21)

![inode layer和log layer的write()系统调用流程](https://github.com/user-attachments/assets/a5feb251-9d0c-4ddb-9420-e18004438888)

* readi()读取inode所指中的block内容至指定内存中。具体是根据指定的cache在icache中的inode中的block#，读取block内容至buffer cache中，再把buffer cache中内容复制到传入地址所指内存中。
* writei()将指定内存中的内容写到buffer cache中。具体是根据指定的cache在icache中的inode中的block#，读取block内容至buffer cache中，再把传入地址所知内存内容复制到buffer cache，以备后续log日志系统写入磁盘。

```c
// Read data from inode.
// Caller must hold ip->lock.
// If user_dst==1, then dst is a user virtual address;
// otherwise, dst is a kernel address.
int
readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return 0;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyout(user_dst, dst, bp->data + (off % BSIZE), m) == -1) {
      brelse(bp);
      tot = -1;
      break;
    }
    brelse(bp);
  }
  return tot;
}

// Write data to inode.
// Caller must hold ip->lock.
// If user_src==1, then src is a user virtual address;
// otherwise, src is a kernel address.
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      n = -1;
      break;
    }
    log_write(bp);
    brelse(bp);
  }

  if(n > 0){
    if(off > ip->size)
      ip->size = off;
    // write the i-node back to disk even if the size didn't change
    // because the loop above might have called bmap() and added a new
    // block to ip->addrs[].
    iupdate(ip);
  }

  return n;
}
```

#### 8.4.2 系统调用对inode的使用

![ialloc()对disk inode 和icache inode的分配](https://github.com/user-attachments/assets/92b22e8e-76cb-481d-84ef-55bf9f119e39)

![exec()中对inode的使用](https://github.com/user-attachments/assets/5d03bd2c-6b84-4c49-8e88-19663fa00b35)

![exit()对inode使用完成](https://github.com/user-attachments/assets/9070aad9-c92b-497b-af53-7ab6473b6530)

* ialloc()进行对disk inode和icache inode的分配。
* iget()进行对icache inode的引用和获取（如果没有则分配）。
* iput()进行对icache inode的减引用和写回disk inode（如果inode refcnt未0就写回磁盘）。
* 期间所有系统调用使用的inode都是icache中的inode，因此上面exec()、read()、write()系统调用中readi()和writei()对inode的读取都是使用的是icache inode。

### 8.5 directory layer
#### 8.5.1 获取指定路径文件的inode过程

![获取指定路径文件的inode过程](https://github.com/user-attachments/assets/d7802d69-c21b-4a94-8ea1-b3eb0123829f)

### 8.6 file descriptor layer
#### 8.6.1 xv6 fd实现
```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};

struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe; // FD_PIPE
  struct inode *ip;  // FD_INODE and FD_DEVICE
  uint off;          // FD_INODE
  short major;       // FD_DEVICE
};

struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;
```
每个进程结构体中一个打开的文件file指针数组，而fd文件描述符就是这个数组的索引下标。

![文件结构体以及文件描述符的分配](https://github.com/user-attachments/assets/94e18412-5f84-4363-9d9e-0285c49cb21b)

```c
uint64
sys_read(void)
{
  struct file *f;
  int n;
  uint64 p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argaddr(1, &p) < 0)
    return -1;
  return fileread(f, p, n);
}

// Fetch the nth word-sized system call argument as a file descriptor
// and return both the descriptor and the corresponding struct file.
static int
argfd(int n, int *pfd, struct file **pf)
{
  int fd;
  struct file *f;

  if(argint(n, &fd) < 0)
    return -1;
  if(fd < 0 || fd >= NOFILE || (f=myproc()->ofile[fd]) == 0)
    return -1;
  if(pfd)
    *pfd = fd;
  if(pf)
    *pf = f;
  return 0;
}

// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  if(f->readable == 0)
    return -1;

  if(f->type == FD_PIPE){
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  } else if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
  } else {
    panic("fileread");
  }

  return r;
}
```

#### 8.6.2 pipe(p)管道内核实现

## 9 Concurrency revisited

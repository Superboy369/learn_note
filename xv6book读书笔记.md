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
## 7 Scheduling
## 8 File system
### 8.1 buffer cache layer
buffer cache 作为外存（磁盘、固态硬盘）在内存中的缓存，其实现保证了：
* 多个进程对同一块 disk block 访问的互斥性。
* 根据程序的局部性原理缓存最近可能使用的 disk block ，减少系统对外存的访问，提高访问效率。
#### 8.1.1 bread()
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
#### 8.1.2 bwrite()
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
#### 8.1.3 buffer cache LRU 替换策略的实现
* 在 xv6 中使用双向链表实现的 buffer cache 的 LRU 策略，双向链表中头部代表最近刚刚使用完，尾部表示最近最少使用。
* 双向链表在 xv6 内核启动时初始化好。
  * 然后在 bread() 调用 bget() 时，如果已经 cache 过，就将 refcnt 加一后返回上锁之后的 buffer cache，如果没有 cache 过，就按照 LRU 策略从后往前扫描双向链表，获取第一个 refcnt == 0 的 buffer cache 上锁之后返回。
  * 在 caller 使用完 buffer cache 之后调用 brelse() 时，brelse() 会将 refcnt 减一之后，判断 refcnt 是否为0，如果不为0什么也不干直接释放锁，如果为0说明没有一个进程在继续使用这个 buffer cache 了，所以把它移至双向链表头部表示最近经常使用。
#### 8.1.4 buffer cache 中的锁
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
### 8.2 logging layer

![未命名文件.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/471eea40d3e7459fb4db23cbef281d4c~tplv-73owjymdk6-jj-mark:0:0:0:0:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTUyNjU3NDgzNTg0MjU2NyJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1722871301&x-orig-sign=5wx4fsFto2QsVyK18algFHEN%2FC8%3D)
## 9 Concurrency revisited

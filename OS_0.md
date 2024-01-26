# 1. 基础部分

## 1.1 从CPU上电到进入OS

- 计算机上电的时候干了啥
	因为计算机是取值执行的, 一上电, CPU reset引脚使能, CPU初始化内部寄存器, X86CPU是把存指令地址的寄存器置初始值, 然后PC寄存器开始往MAR送地址, 开始取指令执行.

总体上来看:
PC电源上电之后, 80x86结构CPU会自动进入`实模式`, 并从地址0xFFFF0开始自动执行程序代码,这个地址通常是`ROM-BIOS中的地址`. PC的BIOS将执行一些`系统的自检`, 并`从物理地址0`处开始`初始化BIOS中断向量`. 然后, 它把可启动设备的`第一个扇区(磁盘引导扇区,512Byte)`读入`内存物理地址0x7c00处`, 并跳到这个地方. 启动设备通常就是软驱或者硬盘. 

Linux的最最前面部分是用8086汇编写的`bootsect.s`, 它将`由BIOS读入`到内存绝对地址`0x7C00(31KB)处`, 当它被执行就会`把自己移动`到内存物理地址`0x90000(576KB)处`, 并把启动设备中的`后2KB字节代码(setup.s)`读取到内存`0x90200`处, 而`内核的其他部分(system模块)`则被读入到内存地址`0x10000(64KB)开始处`, 因此从机器上电开始顺序执行的程序如图:

![](assets/Pasted%20image%2020230310153601.png)

因为当初system模块长度不会超越0x80000字节(512KB), 所以bootsect.s先把自己的代码移到0x90000处, 然后从内存地址0x10000开始, 放system模块, 也不会覆盖0x90000以后的bootsect跟setup模块.
随后setup程序会把system模块移动到物理内存0开始处. 随后进入保护模式并跳到0x0000处, 从system模块的head开始执行. 此时, 所有32位运行方式的设置启动被完成: IDT, GDT和LDT被加载. 处理器和协处理器也被确认, 分页工作也设置好了; 最后调用main.c中的main(). 上面就是head.s中完成的, 这部分至关重要, 错一点系统就起不来.

- 启动引导内核在内存中的位置和移动后的位置:
![](assets/Pasted%20image%2020230310154109.png)

- 上电自动执行rom code(ROM BIOS)
![](assets/Pasted%20image%2020230309122016.png)

CS:IP 置完初值后, 检查相关硬件设备, 开始执行rom code读引导扇区的代码(磁盘0磁道0扇区)到内存0x7c00处. 所以把CS重定向到0x07c0 IP:0x0000, 待左移4位, 就是0x7c00.

自己手动写操作系统, 肯定是要从引导扇区开始, 从OS所在的磁盘位置, 把OS的代码读到内存再执行, 自此开始进入OS.

- 0x7c00处存放的代码
![](assets/Pasted%20image%2020230309123424.png)

- 引导扇区的代码 bootsect.s
```asm
.globl begtext, begdata, begbss, endtext,enddata,endbss
.text //文本段
begtext:
.data  //数据段
begdata:
.bss   //未初始化数据段
begbss:
entry start //关键字entry告诉链接器 程序的入口在哪
start:
mov ax,#BOOTSEG   //BOOTSEG = 0x07c0
mov ds,ax  //ds段寄存器 = 0x07c0 寻址就会先左移4位,去操作0x7c00的数据
mov ax,#INITSEG   //INITSEG = 0x9000
mov es,ax         //SETUPSEG = 0x9020
mov cx,#256  //从这里开始把07c0:0000的256个字节移动到9000:0000处.        
sub si,si
sub di,di
rep movw
jmpi go,INITSEG //go地址赋值给ip, INITSEG赋值给cs, 间接跳转
```
rom code指令让CPU从引导扇区读入512个字节放到0x7c00开始的内存空间, bootsect.s把从0x7c00开始的256字节移动到0x9000:0000处. 然后跳转到9000:go这个位置开始执行.

- jmpi go, INITSEG 后的世界:
![](assets/Pasted%20image%2020230309130350.png)

上面bootsect.s把内存地址从0x7c00开的256字节移到0x90000开始的内存. go: ds段基地址设置到0x90000, es段跟ss段都是0x90000, sp寄存器是0xff00. 
sload_setup: 读取磁盘扇区, ax中的ah为操作类型, al为扇区数量, cx中的ch为柱面号,cl为开始的扇区,dx中的dh为磁头号, dl为驱动器号, 都设置好, 然后 int 0x13 申请中断程序.
从第2个扇区开始, 0号磁头, 0号驱动器读取setup. 读到哪里呢, 读到`es:bx==9000:200`, 即0x90200内存处.
`jnc ok_load_setup` : 不进位就跳转到ok_load_setup.

ok_load_setup:
读取setup代码到0x90200后, 再把system模块读到内存0x10000处. 因为后面的程序还需要用0地址的BIOS中断向量, 所以暂时不能覆盖到0x0000的位置.
![](assets/Pasted%20image%2020230309145933.png)

- read_it:
![](assets/Pasted%20image%2020230309151817.png)

read_it读完system模块(os代码). 就可以jmpi 0,SETUPSEG 到9020:0000进行执行.

- setup.s:
前面bootsect先把system模块读到0x10000内存处了, 现在setup把os代码移动到0地址开始的内存, (所以前面需要把0x7c00的代码, 往后移动到0x90000的地方). 后面进入实模式, 继续执行, OS一直在0地址开始的内存区, 往上是应用程序. 
![](assets/Pasted%20image%2020230309152528.png)

扩展内存大小是 最初PC最多只有1M的内存, 1M往后全是属于扩展内存. 操作系统对内存进行管理必须知道内存实际有多大.  

setup准备进入保护模式:
![](assets/Pasted%20image%2020230310162235.png)

32位运行方式的设置启动被完成: IDT, GDT和LDT被加载. 

setup进入保护模式
从此刻开始, 就不再是实模式了. CS只用于全局描述表GDT. cr0寄存器中, 第0位, 0是实模式, 1就是保护模式.
![](assets/Pasted%20image%2020230310162505.png)

介绍保护模式下的地址解释和中断处理
![](assets/Pasted%20image%2020230310162634.png)

setup跳转到0地址的system中的head:
所以cs 是8 ip 是0, 查完得到的地址就是0地址.
![](assets/Pasted%20image%2020230310163719.png)

- 内核代码的image
![](assets/Pasted%20image%2020230310164718.png)

head.s :
![](assets/Pasted%20image%2020230310174003.png)

head.s 又重新初始化GDT IDT 因为setup的初始化GDT是为了进入保护模式, 进而跳转到system的head, 此时GDT是临时用的. head.s 重新初始化, 是为了OS真正要用的.

![](assets/Pasted%20image%2020230310182934.png)

jmp after_page_tables:
![](assets/Pasted%20image%2020230310183210.png)

把main函数的参数, 跟函数地址`_main`地址压栈, 即将进入main函数.

main.c:
终于进入c语言代码了, 不用看汇编那么累. 但内核代码里有大量内联汇编的.
都是以init结尾的, 所以, main.c做的就是初始化.
![](assets/Pasted%20image%2020230310184938.png)

只看一个例子:
end_mem >>= 12; 4K 一页. 内存有多大, 从CMOS里读出硬件信息, 然后一路传递到这里. 前面used的是os部分.
![](assets/Pasted%20image%2020230310185232.png)

启动先到这, 后面的内容可以看linux内核完全注释.

## 1.2 操作系统接口

连接两个东西, 信号转换, 屏蔽细节..
![](assets/Pasted%20image%2020230310214235.png)
![](assets/Pasted%20image%2020230310214702.png)
![](assets/Pasted%20image%2020230310214746.png)
![](assets/Pasted%20image%2020230310220457.png)
![](assets/Pasted%20image%2020230310220532.png)

总结: 操作系统接口, 就是内核向应用层提供的软件服务.  应用层的程序, 调用了操作系统内核中实现的函数,  就叫做系统调用. (system call) 操作系统接口, 就是系统调用.

![](assets/Pasted%20image%2020230310220909.png)

比如c语言的`fopen`函数, 它调用了文件系统提供的`open`这个函数, `printf` 调用了`write`这个系统函数, 都是C标准库进行系统调用的例子.

系统调用六大类:
![](assets/Pasted%20image%2020230316164948.png)

## 1.3 系统调用的实现

内核区跟用户区, 都在内存里, 那为什么用户态的程序要调用内核态的接口,为什么不能直接跳过去呢.
![](assets/Pasted%20image%2020230310221749.png)

为什么用户态不能随便访问内核态

内存分成了两个大块, 一块是专门给内核用的, 一块是专门给用户用的. 区分用户态和内核态是一种处理器的硬件设计干的. 硬件设计所支持的特权级, 使得用户态的程序,没法随意进入内核态.
![](assets/Pasted%20image%2020230310222548.png)

>特权级可以参考[一个操作系统的实现](https://dbb4560.github.io/2019/10/29/Orange'S-%E4%B8%80%E4%B8%AA%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%AE%9E%E7%8E%B0-%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E7%89%B9%E6%9D%83%E7%BA%A7/).

那用户态如何进入内核态
x86提供了 特定中断, 才能进入内核态.
![](assets/Pasted%20image%2020230312123137.png)
![](assets/Pasted%20image%2020230312123437.png)

Linux系统调用的实现细节:
![](assets/Pasted%20image%2020230312123704.png)
![](assets/Pasted%20image%2020230312124434.png)

IDT就是上图底下的那个表格.
应用层`printf` -> 库函数`printf`->库函数`write`->系统调用`write`->内核宏`_syscall3`-> 内核中断`0x80`, `__NR_##name`(宏定义系统调用索引标号为4) -> 中断处理宏`set_system_gate` -> `_set_gate`   -> CS=8, IP=&`system_call`, 且CPL低2位为0. -> `_system_call_table`(函数指针数组)去找函数入口地址
![](assets/Pasted%20image%2020230312125744.png)
![](assets/Pasted%20image%2020230312130014.png)
![](assets/Pasted%20image%2020230312130433.png)

实验二:
- 在include/unistd.h中添加 `__NR_whoami` 跟 `__NR_iam`
- kernel/system_call.s 中 nr_system_calls改一下
- include/linux/sys.h 添加extern int sys_iam();和 extern int sys_whoami(); 
	`sys_call_table[]` 也要添加函数名
- kernel/中新建一个who.c , Makefile改一下.

![](assets/Pasted%20image%2020230313210632.png)
![](assets/Pasted%20image%2020230313210645.png)
![](assets/Pasted%20image%2020230313210721.png)

# 2. 进程与线程

## 2.1 CPU管理的只管想法

操作系统在管理CPU资源的时候, 才引入了多进程. 所以在进入进程之前, 先来看看CPU 的管理.
![](assets/Pasted%20image%2020230313211117.png)
![](assets/Pasted%20image%2020230313211347.png)
![](assets/Pasted%20image%2020230313211813.png)

问题在于: IO指令非常的慢. 所以CPU只能等待IO操作, 等IO拿到数据再继续. 所以CPU就这样走走停停. 利用率50%.
所以在IO的时候, cpu切到别的任务.就行了, 等IO操作结束, 发出中断, CPU再回来.
![](assets/Pasted%20image%2020230313212612.png)

这就是多到程序交替执行.
![](assets/Pasted%20image%2020230313212630.png)
![](assets/Pasted%20image%2020230313213016.png)

并发执行, CPU的效率就高了, 那么如何让CPU并发执行?
给PC切换地址执行, 就行了吗? 执行中的程序跟静态的程序是不一样的.
执行程序1的时候, 会产生一些数据, 是跟程序相关的关键信息. 当切换到程序2的时候, 必须要把程序1的这些关键信息保存下来, 以便于等会切回来的时候好复原现场. 执行程序2的时候也一样, 所以, 每个程序都要记录.
记录信息的东西, 就是PCB(Process Control Block), 这是个结构体.
![](assets/Pasted%20image%2020230313213226.png)

进程来了.  进行中的程序.
![](assets/Pasted%20image%2020230313213555.png)

## 2.2 多进程图像

![](assets/Pasted%20image%2020230313214552.png)

用户看到的是三个不同的进程(任务). 而OS就负责管理和记录这三个进程.
![](assets/Pasted%20image%2020230313214909.png)

多进程是OS的核心.

第二部分, 多进程图像如何组织
![](assets/Pasted%20image%2020230313215738.png)

OS感知和管理, 操作进程, 就靠PCB. 上图假设只有一个CPU. 但现在的机器实际上有多个Core, 就是说可以同时运行多个进程. 多核后面再说.
进程有多个, 那有在执行中的, 就有在等待的, 还有在等待某个事件的.
所以就可以根据状体把程序分类:

![](assets/Pasted%20image%2020230313220208.png)

```c
// 任务（进程）数据结构
struct task_struct {
/* these are hardcoded - don't touch */
    long state; /* -1 unrunnable, 0 runnable, >0 stopped */         // 任务的运行状态（-1 不可运行，0 可运行(就绪)，>0 已停止）
    long counter;    //counter 值的计算方式为 counter = counter /2 + priority,  优先执行counter最大的任务;  任务运行时间计数(递减)（滴答数）(时间片)
    long priority;      // 运行优先数。任务开始运行时 counter = priority，越大运行越长
    long signal;        // 信号。是位图，每个比特位代表一种信号，信号值=位偏移值+1
    struct sigaction sigaction[32];     // 信号执行属性结构，对应信号将要执行的操作和标志信息
    long blocked;   /* bitmap of masked signals */  // 进程信号屏蔽码（对应信号位图）
/* various fields */
    int exit_code;                                                  //任务执行停止的退出码，其父进程会取
    unsigned long start_code,end_code,end_data,brk,start_stack;     // 代码段地址,代码长度，数据长度，总长度，栈段地址
    long pid,father,pgrp,session,leader;                            // 进程标识号，父进程号，父进程组号，会话号，会话首领
    unsigned short uid,euid,suid;                                   // 用户标识号， 有效用户id，保存的用户id
    unsigned short gid,egid,sgid;                                   // 组标识号，有效组号，保存的组号
    long alarm;                                                     // 报警定时器值
    long utime,stime,cutime,cstime,start_time;                      // 用户态运行时间，系统态运行时间，子进程用户态运行时间，子进程系统态运行时间，进程开始运行时刻
    unsigned short used_math;                                       // 是否使用了协处理器
/* file system info */
    int tty;        /* -1 if no tty, so it must be signed */    // 进程使用 tty 的子设备号。-1 表示没有使用
    unsigned short umask;                                       // 文件创建属性屏蔽位
    struct m_inode * pwd;                                       // 当前工作目录i节点
    struct m_inode * root;                                      // 跟目录i节点
    struct m_inode * executable;                                // 执行文件i节点
    unsigned long close_on_exec;                                // 执行时关闭文件句柄位图标志
    struct file * filp[NR_OPEN];                                // 进程使用的文件表结构，用于保存文件句柄
/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
    struct desc_struct ldt[3];                                      // 局部描述符段， 0-空，1-代码段 cs，2-数据和堆栈段 ds&ss
/* tss for this task */
    struct tss_struct tss;                                      // 本进程的任务状态段信息结构
};
```
![](assets/Pasted%20image%2020230313222031.png)

启动磁盘读写, 然后把自己的状态置为阻塞态, 放到磁盘等待队列里, 然后调用`schedule()`. 这个函数很核心.  其中`getNext()` 就是调度过程, 选好下一个进行, 然后`switch_to` 切换到下个进程.  操作的数据就是PCB结构体类型.
![](assets/Pasted%20image%2020230314114403.png)

(哈工大的OS课是讲OS是如何运作起来的. 而非纯OS理论, 非文科OS.)
![](assets/Pasted%20image%2020230314120408.png)

上面这幅图就是切换PCB, 就是切换进程的过程. 这个过程是个精细活, 需要用汇编来写. 如图内联汇编. 需要懂X86的体系结构, 寄存器的用法.
```asm
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__("cmpl %%ecx,_current\n\t" \
    "je 1f\n\t" \
    "movw %%dx,%1\n\t" \
    "xchgl %%ecx,_current\n\t" \
    "ljmp %0\n\t" \
    "cmpl %%ecx,_last_task_used_math\n\t" \
    "jne 1f\n\t" \
    "clts\n" \
    "1:" \
    ::"m" (*&__tmp.a),"m" (*&__tmp.b), \
    "d" (_TSS(n)),"c" ((long) task[n])); \
}
```
![](assets/Pasted%20image%2020230314120633.png)

多进程交替执行 的时候, 对内存中的数据是怎么保护的. 不然进程1跑着把某个内存数据改了, 这个数据在进程2正在使用, 进程2就会出错, 整个进程就会崩掉.. 
如何对地址100的数据进行限制读写呢?
![](assets/Pasted%20image%2020230314120954.png)

可以通过映射表. 是内存管理的部分. 使用映射表, 那么100地址就不再是真实的地址100了. 根据映射表, 100可以被映射到其他内存区域. 这就是进程的内存保护.
![](assets/Pasted%20image%2020230314121226.png)

每个进程在自己的文本中, 相同内存地址作操作数, 经过映射表, 实际操作的是不同的物理内存, 就很好的保护了各个进程使用的数据.
各个进程用的内存被保护起来了, 那么多个进程能不能合作处理一段数据呢?
![](assets/Pasted%20image%2020230314121623.png)

可以的, 但是需要安排好. 比如上图进行打印工作, 由于是交替执行, 如果进程1正在往地址7写数据, 还没写完呢, 进程2就也写了, 那就乱了. 所以需要安排好, 等进程1写完, 进程2再往下一个地址区写.
这个问题比较经典的就是生产者消费者的例子:
![](assets/Pasted%20image%2020230314123421.png)
![](assets/Pasted%20image%2020230314124059.png)

程序是交替执行的, CPU乱序执行, 所以很可能不是等生产者执行完, 消费者再执行. 而是如上图右边. 消费者的counter实际是4.
如何避免这个问题.  `进程同步`.
![](assets/Pasted%20image%2020230314124430.png)
![](assets/Pasted%20image%2020230314124752.png)

> 进程切换 = 切换指令流 + 切换资源

## 2.3 用户级线程

- 为何讲线程
![](assets/Pasted%20image%2020230314125453.png)

进程1执行到write IO操作, 执行不下去了, 要切换函数执行. 那我们可不可以只切指令, 不切映射表.
线程thread被引入. linux0.11的时候还没支持线程.
![](assets/Pasted%20image%2020230314141509.png)

资源还是那块给进程分配的资源, 只是切换指令, 映射表不用变, 只是把PC寄存器变一下. 不用保存很多东西. 开销小. 线程在进程内部来回切换.
可以说, 进程切换= 指令切换+映射表切换(资源切换). 需要学完内存管理的部分才能结合起来. 这就是分治的思想.
线程有什么价值呢?
![](assets/Pasted%20image%2020230314144452.png)

浏览器浏览网站肯定要去服务器下载网页, 浏览器渲染, 就能看到了. 
所以就会有下载线程, 跟渲染线程. 你不能等所有内容都下完再一起显示, 而是边下边显示.
下载一部分, 切换到渲染线程, 先让用户能看到内容. 再切回去继续下载.
线程切换实例:
![](assets/Pasted%20image%2020230314145141.png)

创建俩线程, 一个下载数据, 一个显示内容. 要想交替执行, 就要在函数代码中, 写相关代码. 调用函数(Yield)切到另一个线程.
![](assets/Pasted%20image%2020230314145613.png)

任何复杂程序, 只要在脑海中有图, 然后用编程语言实现就好了. 
切换过程:
![](assets/Pasted%20image%2020230314150556.png)

- A执行中, 调用B, 跳到B执行, 返回地址压栈.
- B中执行, 调用Yield函数. 跳到Yield函数体内执行, yield负责跳转300地址.
- C执行, 调用D. D调用yield, 跳到B(), 204, B结束执行ret.
- B执行完应该返回到A里的104地址继续执行, 但是却不是. 因为两个线程的返回地址都压在了一个栈里, ret是把404pop执行了, 乱套.
假如两个线程共用一个栈, 导致函数返回时, 出问题. 所以要用两个栈(开辟两个栈帧):
![](assets/Pasted%20image%2020230314151307.png)

每个线程各自有个栈. 所以需要有个地方保存栈的地址. TCB结构体负责保存.
![](assets/Pasted%20image%2020230314151932.png)
![](assets/Pasted%20image%2020230314152458.png)

用户态的线程切换.
![](assets/Pasted%20image%2020230314152654.png)

用户态的问题在于, 假如下载线程中一旦网卡阻塞, 内核就会切换到别的进程, 进程1的剩余线程就不会执行了. 浏览器还是什么都不显示. 
内核态的线程
![](assets/Pasted%20image%2020230314153016.png)

TCB在内核, 都在进程1中, 下载线程阻塞, 就会切到显示线程继续执行. 所以内核态的线程并发性更好.

## 2.4 内核级线程

一般来说, OS下用户级线程, 内核级线程, 跟进程并存, OS调度才足够灵活. 进程一定在内核中.
核心级线程
![](assets/Pasted%20image%2020230314170602.png)

如果内核不支持核心级线程, 多核心就没用, 因为多核不是多处理器, `多核共用Cache和MMU`.
多线程到内核中, 才能充分利用多核. 若是多进程, MMU也得切来切去, 发挥不了多核优势.

> 并行与并发不同. 并行是线程A执行的时刻, 线程B也同时执行.  并发是同时间段内执行, 但是交替执行.

多个内核级线程, 可以让多核同时并行执行.  而用户级线程, OS看不到, 无法分配硬件核心. 
多进程, 跟用户级线程都无法发挥多核的优势.  核心级线程的必要性就在于此.

![](assets/Pasted%20image%2020230314171342.png)

用户栈与内核栈, 是两个内存区域. 内核往往跑在最高的物理内存地址上.
线程既能在用户栈跑, 也能在内核栈跑. 
内核在切换线程的时候, 内核栈跟用户栈一起切换.
用户级线程切换是TCB切, 根据TCB切换用户栈, 即可. 核心级线程切换, TCB切换, 再根据TCB切换一套栈(用户栈和内核栈).
![](assets/Pasted%20image%2020230314191156.png)

用户态产生INT, 硬件就会启动内核栈, 根据寄存器中的信息, 找到线程的内核栈. 把刚刚用户级线程的用户栈的SS跟SP压栈到内核栈, 还要把EFLAGS 跟原来的PC,CS压栈.(完全把用户态的栈帧, 跟标志寄存器, 用户态返回地址保存到内核栈的栈帧之内.)
IRET中断返回: 内核栈pop, 弹回去用户栈继续执行.
![](assets/Pasted%20image%2020230314191746.png)

到read() 就开始陷入内核态. 继续压入返回地址304(IP), 跟CS(段基地址)(这里从内核栈到用户栈还是一个栈到另一个栈, 要回用户栈的时候,就把304跟CS pop就会继续执行CS:304了). 然后就会调用system_call. 找到具体的中断函数, 调用. 然后开始执行.
![](assets/Pasted%20image%2020230314192109.png)

执行sys_read(), 阻塞. 找下一个等待的线程, 切换线程. switch_to的参数是TCB类型. 
把cur的esp保存, 然后把next->esp赋值给esp寄存器, 切到线程T的内核栈了, 线程T的内核栈与线程T的用户栈相关联.. ret弹栈, 执行. 然后再弹CS PC, iret中断返回, 回到用户态程序线程T执行.
![](assets/Pasted%20image%2020230314194901.png)

线程S内核栈中是???是sys_read()函数中的某个位置. 线程T的内核栈, PC跟CS是线程T用户栈进入中断前的指令地址, ????是在线程T内核态函数中, 有iret的代码.
总结一下内核线程切换:
![](assets/Pasted%20image%2020230314195850.png)
![](assets/Pasted%20image%2020230314202103.png)
![](assets/Pasted%20image%2020230314202449.png)

接下来看源码. 

## 2.5 内核级线程实现







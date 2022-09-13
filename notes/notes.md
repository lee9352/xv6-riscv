# bootloader
bootloader由qemu提供
执行完毕之后，内核代码被搬至0x80000000处
然后开始执行一段汇编代码entry.S

# entry.S
在start.c中定义了一个栈stack0[4096 * NCPU]，每个cpu可用的栈大小为4096 Bytes
这里会把stack0的地址放入sp寄存器
然后给sp+4096,即sp现在指向stack0中给cpu0的栈底(栈是从高地址向低地址增长的)
然后跳转到start.c执行

# start.c
设置mstatus寄存器，把mpp置为1，这意味着执行mret时会进入s mode(当前是m mode)
设置mepc寄存器，置为main函数的地址，意味着mret会跳转到main函数

之后还设置了s mode的中断寄存器sie

初始化处理器时钟中断timer_init()

这里指定了mtime和mtimecmp寄存器的值，这两个寄存器地址是由操作系统自己定义的
然后把这两个地址记录在scratch寄存器

当mtime的值大于mtimecmp时，触发mtime中断

这里会设置时钟中断函数的入口,把timervec()的地址写入mtvec寄存器

最后通过mret，退出m mode，同时进入main()函数

而这个timervec()里面实际上干的事情是：
- 先交换mscratch和a0寄存器(mscratch存的是数组timer_scratch的地址，该数组在时钟中断时可以临时保存寄存器)
- 把$a1,$a2,$a3临时保存到timer_scratch中
- 取出mtimecmp的地址和interval的值(保存在timer_scratch[3]和[4]中)
- 通过上一步的地址取出mtimecmp
- 给mtimecmp+interval，再放回去，作为下一次触发时钟中断的条件
- 设置一个软中断(把sip的ssip位置为1)**不知道软中断会跳转到哪,刚开始还没有设置sstatus，软中断不会触发**
- 恢复$a1,$a2,$a3和$a0寄存器，同时把timer_scratch的地址又放回$mscratch中
- mret退出timervec()

实际上，如果当前处于用户态，时钟中断发生后，timervec()设置好ssip位，mret之后马上会跳转到usertrap去了(通过编写了一个loop.c死循环得以验证)
但是如果系统启动后，啥也不做，那么最后会执行一个sh进程，此时会发现timervec()永远不会从用户态进入，而都是从内核态进入的
因为sh进程会readcmd,这个函数会执行sys_read系统调用
这个调用会通过consoleread()读取数据，如果没有数据，就sleep,该进程就一直没被调度，之后都是内核在瞎几把跑
然后内核经常会关闭sie，于是mret之后不会马上进入kerneltrap，还需要等开启中断之后才行


consoleread()的数据又从哪来?
里面有一个buffer叫cons.buffer
当uart有输入时，会进入一个uartintr中断处理函数，会把uart寄存器里面的数据通过函数consoleintr()写入cons.buffer

但是如果uart一直没有数据进来，consoleread就会被sleep



usertrap()和kerneltrap()会通过devintr()
这个函数判断是时钟中断还是设备中断
给tick++ **这里不知道是在干嘛**

为了减少硬中断的处理时间，把中断处理分上半部和下半部。上半部需要立即处理，且此时会屏蔽中断。
下半部就是软中断，负责运行那些可以慢一点执行的任务

软中断还可以用于进程内部，运行在不同核心线程之间通信


# main.c
初始化各种配置
## concole_init()设备初始化
初始化了一个串口uart_init()
设备的输入和输出都是通过串口完成的

再把read和write系统调用连接至console

当执行printf函数时，该函数调用consputc，即把字符通过设备输出
而consputc又把字符传递给uart，而qemu通过uart读取到字符，就能显示输出了

### uart_init()
串口地址为0x0x10000000
注：**调试这里的代码时，gdb不能正确显示内存变化情况**
里面有很多寄存器，暂时不清楚他们的含义

## kinit()
初始化内核可用内存，范围从内核data结束的位置开始，一直到PHYSTOP(物理内存结束的位置
)
从地址0x80026000一直到0x88000000，以页为单位

每一页的首地址用于存放指针，指向下一个可用的页

例如：0x80028000的值为0x80027000(栈从高往低增长)

用一个struct kmem链表记录空闲页，链表的next指针都位于每一页的页首，指向的位置为下一个可用页

## kvminit()
创建内核最高级(L2)页表，并返回内核L2页表地址

从可用物理地址中，用kalloc()函数查询kmem空闲链表，取一页用于存放内核页表
kpgtbl = 0x87fff000,这是内核栈初始化之后的第一个可用页面,用来作为L2页表

在页表内，通过kvmmap()函数给uart,mmio,plic等设备地址和kernel text,data区的地址做地址映射，修改页表项(这一部分va和pa一样)
trampoline和kernel stack也要做映射(va和pa不一样)

### kvmmap() && mappages()
kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
kpgtbl为内核的L2页表
perm表示权限
  a = PGROUNDDOWN(va); //表示所需空间的第一页位置
  last = PGROUNDDOWN(va + size - 1); //表示所需空间最后一页位置
如果只分配一页，那么a == last

函数walk负责找到虚拟地址va对应的页表项

#### walk()寻找到该va对应的0级页表项
riscv采用sv39分页机制

walk()第一个参数为kpgtbl，就是内核L2页表地址

walk()内部有一个for循环，循环2次

第一次循环根据va和给定的L2页表，寻找记录L1页表位置的页表项
寻找方式为:把va>>9*2+12bit = ppn2 = 0,kpgtbl [ ppn2 * size(pte) ]即为该地址在L2页表的页表项
检查该页表项的v位是否为1，如果不为1，则会调用kalloc分配一页作为页表

第一次循环结时，会分配新一页0x87ffe000,作为1级页表
并把该页表地址存入页表项

第二次循环时
va>>9*1+12bit = ppn1 = 0x80, pagetable_L1 [ ppn1 * size(pte) = 0x400] = 0x87ffe400 为该地址在L1的页表项
此时该页表项的v位为0，仍需分配一页作为L0页表

分配0x87ffd000作为0级页表，同时把该地址存入刚刚的页表项

得到0级页表后，找到该虚拟地址va在0级页表中的页表项
即可返回该页表项的地址

对于内核空间，大部分虚拟地址和物理地址一样，但是trampoline不一样
trampoline的va是MAXVA-PGSIZE(0x3ffffff000),但是pa为KERNBASE+offset(KERNBASE是内核代码存放的地方)



### proc_mapstacks()
用于给每个进程分配一个内核栈
先给每个进程分配一个单独的va
这个va是从trampoline开始向下增长的

内核能看到所有进程的va各不相同

pa还是从之前分配的内存空间中寻找
然后用kvmmap分配页表

## kvminithart()
把内核L2页表地址写入satp寄存器

以上完成了页表系统初始化设置

## procinit()
把之前给每个进程分配的内核栈的虚拟地址va
记录到每个进程pcb中

## trapinit()
初始化了一把锁tickslock

## trapinithart()
把kernelvec()函数的入口写入$stvec寄存器

kernelvec()是处于内核态时，处理中断或异常的代码
保存当前寄存器，然后跳转到kerneltrap()函数

kerneltrap()判断当前是时钟中断还是设备中断
时钟中断就让出cpu，内核居然也会让出cpu，很奇怪
**查看了一下代码，内核让出cpu是指用户进程陷入了内核，但是时间片用完了，还是要让出去**


## plicinit() 设备中断
把uart和virtio0中断设置优先级为1，否则这两个中断就被关闭了
这两信号有专门的内存地址
就往这两个地址写1就行了

但是gdb查看内存值有点问题
得用qemu查看内存

## plicinithart()
给每个cpu core设置硬件中断enable

## 后面就是配置文件系统

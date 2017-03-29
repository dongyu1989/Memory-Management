# Memory-Management
内存地址
======
内存地址(memory address)：
作为访问内存单元的一种方式，但是，当使用80x86微处理器时，必须区分以下三种不同的地址：

逻辑地址
------
  包含在机器语言指令中用来指定一个操作数或一条指令的地址。这种寻址方式在80x86著名的分段结构中表现得尤为具体。它促使MS-DOS或Windows程序员分成若干段。每一个逻辑地址都是由一个段(segment)和偏移量(offset或displacement)组成，偏移量指明了从段开始的地方到实际地址之间的距离。
  
线性地址(又称虚拟地址)
------
物理地址
-------
用于内存芯片级内存单元寻址。他们与从微处理器的地址引脚出发到内存总线上的电信号相对应。物理地址由32位或36位无符号整数表示。

内存控制单元(MMU)通过一种称为分段单元(segmentation unit)的硬件单路把一个逻辑地址转换成线性地址;接着，第二个称为分页单元（paging unit）的硬件电路把线性地址转换成一个物理地址。
![image](https://github.com/wangdongyu1989/Memory-Management/blob/master/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%8420170322a.jpg)
linux内核采用页式存储管理。虚拟地址空间划分成固定大小的“页面”，由于MMU在运行时将虚拟地址“映射”成某个物理内存页面中的地址。与段式存储管理相比，页式存储有很多好处。首先，页面都是国定大小的，便于管理。更重要的是，当要将一部分物理空间的内存换出到磁盘上的时候，在页式存储管理中是按页进行，效率显然要高得多。页式存储管理和段式存储管理所要求的硬件支持不同，一种CPU既然支持页式存储管理，就无需再支持段式存储管理。但是，i386的情况是特殊的。由于i386系列的历史演变过程，它对页式存储管理的支持是在其段式存储管理已经存在相当长的时间以后才发展起来的。所以，不管程序是怎样写的，i386CPU一律对程序中使用的地址先进行段式映射，然后才能进行页式映射。

段映射阶段
========

段选择符和段寄存器
---------------
一个逻辑地址由两部分组成：一个段标识符和一个指定段内相对地址的偏移量。段标识符是一个16位长的字段，成为段选择符(Segment Selector),而偏移量是一个32位长的字段，段选择符如下图。
![image](https://github.com/wangdongyu1989/Memory-Management/blob/master/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%8420170323a.jpg "段选择符")

为了快速方便地找到段选择符，处理器提供段寄存器，段寄存器的唯一目的是存放段选择符。这些段寄存器称为cs,ss,ds,es,fs和gs。但程序可以把同一个寄存器用于不同的目的，方法是先将其值保存在内存中，用完后再恢复。

cs 代码段寄存器，指向包含程序指令的段。

ss 栈段寄存器，指向包含当前程序栈的段。

ds 数据段寄存器，指向包含静态数据或者全局数据段。

其他3个段寄存器作一般用途，可以指向任意的数据段。

cs寄存器还有一个很重要的功能：它含有一个两位的字段，用以指明CPU的当前特权级。值为0代表最高优先级，而值为3代表最低优先级。Linux只用0级和3级，分别称之为内核态和用户态。

![image](https://github.com/wangdongyu1989/Memory-Management/blob/master/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%8420170324a.jpg "进程设置段寄存器")

这里regs->xdss是段寄存器DS的映像，余类推。这里已经可以看到一个有趣的事，就是除CS被设置成USE_CS外，其他所有的段寄存器都设置成USER_DS。这里特别值得注意的是堆栈寄存器SS，它也被设置成USER_DS。就是说，虽然Intel的意图是将一个进程的映像分成代码段，数据段和堆栈段，Linux内核却并不买这个账。在Linux内核中堆栈段和数据段是不分的。

再看看USER_CS和USER_DS到底是什么。那是在include/asm-i386/segment.h中定义的：   
 ![image](https://github.com/wangdongyu1989/Memory-Management/blob/master/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%8420170324b.jpg "USER_CS和DS")

也就是说，Linux内核中只使用四种不同的段寄存器值，两种用于内核本身，两种用于所有的进程。现在，我们将这四种数值用二进制展开并与段寄存器的格式相对照：
首先，TI都是0，也就是说全都使用了GDT。这就与Intel的设计意图不一致了。实际上，在Linux内核中基本上不实用局部段描述表LDT。LDT只是在VM86模式中运行wine以及其他在Linux上模拟运行Windows软件或Dos软件的程序中才使用。

再看RPL，只用了0和3级，内核为0级用户(进程)为3级。在进程的用户空间中运行，内核在调度该进程进入运行时，把CS设置成USER_CS,即0x23。所以，CPU以4为下标，从全局段描述表GDT中找到对应的段描述项。


段描述符
-------
每个段由一个8字节的段描述符(Segment Descriptor)表示，它描述了段的特征。段描述符放在全局描述符表(Global Descriptor Table,GDT)或布局描述符表(Local Descriptor Table, LDT)中。GDT在主存中的地址和大小放在gdtr控制寄存器中，当前正被使用的LDT的地址和大小放在ldtr控制寄存器中。

![image](https://github.com/wangdongyu1989/Memory-Management/blob/master/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%8420170329a.jpg "段描述符定义")

* B0-15 B16-31：基地址全是0，包含段的首字节的线性地址
* L0-15 L16-19：段的上限全是0xfffff，存放段中最后一个内存单元的偏移量，从而决定段的长度。如果G被设置为0，则一个段的大小在1个字节到1MB之间变化；或者，将在4KB到4GB
* G：粒度单位，如果该位清0，则段大小以字节为单元，否则以4096字节的倍数计
* S：系统标志，如果清0，则这是一个系统段，如各类描述表。如果为1，表示一般的代码或者数据段
* TYPE47-40：描述了段的类型特征和它的存取权限

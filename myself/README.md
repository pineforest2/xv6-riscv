这是我学习**xv6-riscv**的仓库，源代码来自[GitHub](https://github.com/mit-pdos/xv6-riscv/)。

我将项目从GitHub上面clone下来之后，将原有的git commit信息全部删除，并重新初始化了该仓库。这貌似造成一个问题，在`.gitignore`文件中是将目录mkfs忽视掉的，导致我的仓库无法跟踪目录mkfs下的文件，主要是`mkfs.c`。
实际上如果你不删除原来的git commit信息，其表现为仅跟踪目录mkfs下的文件`mkfs.c`，其它（新增的）文件是被忽视掉的。
貌似你需要先将`.gitignore`文件中忽视mkfs那一行删除，然后运行`git add ./mkfs/mkfs.c`，运行完之后将还原`.gitignore`文件即可。

## 环境搭建与项目构建

> https://pdos.csail.mit.edu/6.1810/

可去以上MIT的课程网址找到环境搭建的教程。大体上是安装用于交叉编译的gcc和binutils、gdb-multiarch、Qemu。
> xv6-riscv是基于64位RISC-V。

项目的构建工具显然是Make。在安装好上述依赖的软件包之后，使用`make qemu`即可运行项目。

### 调试

- todo

## 代码分析

我们首先打开文件`entry.S`，首先阅读该文件头部的注释可知，每个hart初始都将从地址0x80000000处运行。因此，`kernel.ld`得将代码（实际上是代码汇编得到的节）放到地址0x80000000处。
> hart是RISC-V中的概念，可译为硬件线程。
> `kernel.ld`是针对GCC中链接器ld，称为链接脚本。

接下来在`_entry`中的代码主要是从`stack0`上为每个CPU设置内核栈，具体其注释已经描述地很清楚了。`la`是Load Address，`li`是Load Immediate，`csrr`是Control and Status Register Read。`mhartid`是RISC-V中CSR的一种，其中保存当前hart的ID，一般从0开始。RISC-V中的sp寄存器中保存栈顶地址，其中sp维护的栈是递减的。最后会通过`call`指令调用`start.c`中的`start`函数。
定位到`start`函数，首先进行的是通过设置`mstatus`寄存器，将RISC-V的特权级模式从M到S。
> RISC-V中存在一个特权级模式，也可以叫做特权等级的概念。目前有三种U、S、M，分别对应用户程序模式、监督者模式、机器模式。

其中使用到的函数`r_mstatus`和`w_mstatus`分别是读`mstatus`寄存器和写`mstatus`寄存器，这样的函数命名是简单明了，在xv6-riscv中，这样的函数还有很多，比如接下来的`w_mepc`和`w_satp`等等，它们应该都定义在文件`riscv.h`中，之后便不一一解释了。
函数`start`后面的代码也不分析了，主要是在进行初始化的配置，该函数执行完成会跳转到`main.c`中的`main`函数。
定位到`main`函数，可以看到第0号hart会完成一堆初始化，非第0号hart也有几个初始化需要完成。
我们先来看函数`consoleinit`，该函数所在的文件`console.c`中存在一个全局变量`cons`，具体定义如下。
```c
struct {
  struct spinlock lock;
  
  // input
#define INPUT_BUF_SIZE 128
  char buf[INPUT_BUF_SIZE];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} cons;
```
在`consoleinit`中首先调用函数`initlock`初始化了一把（自旋）锁，那我们接下来就来分析一下xv6-riscv中的自旋锁。
自旋锁对应的结构体`spinlock`在文件`spinlock.h`中，针对其的函数基本都在文件`spinlock.c`中。函数`initlock`就是将结构体中的成员赋值，比较简单。函数`acquire`是获取锁，进入该函数的第一件事情就是调用函数`push_off`，其注释也表明其目的是禁止中断以避免死锁。我们进一步看看`push_off`，首先是通过调用函数`intr_get`得到之前的中断是否使能，然后调用函数`intr_off`禁止中断。剩下的操作都涉及到函数`mycpu`，该函数意思其实很明确，其跟结构体`cpu`和数组`cpus`是相关的。结构体`cpu`定义在文件`proc.h`中。关于该结构体相关的，在文件中都有详细的注释。数组`cpus`定义在文件`proc.c`中，而`NCPU`是一个常量宏，值为8，表示最多能支持8个CPU。
回到`push_off`，最后的几个操作其实都涉及结构体`cpu`中的两个成员`noff`和`intena`，它们应该都是为了嵌套中断服务的，其中`intena`表示是否发生嵌套中断，`noff`表示嵌套的深度。

empty

我们再来看函数`kinit`。在此之前，我们需要了解代码中与RISC-V的内存管理相关的内容。首先是两个常量宏`PGSIZE`和`PGSHIFT`，前者表示一个内存页为4096字节，后者在代码中出现较少，故略去了。
再是两个函数宏`PGROUNDUP`和`PGROUNDDOWN`，二者应该都是将地址作为输入。举个例子说明输出，若输入为5000（这显然是一个十进制表示的地址），则`PGROUNDUP`的输出为8192，而`PGROUNDDOWN`的输出为4096。
同时，在文件`memlayout.h`中，我们可以看到物理内存的布局。目前，我们需要记住的是常量宏`KERNBASE`和`PHYSTOP`，前者的值为`0x80000000L`，而后者的值通过计算应为`0x84000000L`，同时不难得知，该操作系统的内存总大小应为128MiB。
回到`kinit`，跳转所在文件`kalloc.c`，该函数大体上做了两件事情：初始化了kmem锁、并将`end`到`PHYSTOP`之间的内存页free掉（实际就是将这些内存页加入空闲链表）。
下面解释一下`end`是个什么东西，它在`kalloc.c`中如下定义，通过其注释可明白它定义于`kernel.ld`，应该是表示将内核文件加载到内存之后的末地址。
```c
extern char end[]; // first address after kernel.
                   // defined by kernel.ld.
```
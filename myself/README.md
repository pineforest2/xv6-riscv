这是我学习**xv6-riscv**的仓库，源代码来自[GitHub](https://github.com/mit-pdos/xv6-riscv/)。

我将项目从GitHub上面clone下来之后，将原有的git commit信息全部删除，并重新初始化了该仓库。这貌似造成一个问题，在`.gitignore`文件中是将目录mkfs忽视掉的，导致我的仓库无法跟踪目录mkfs下的文件，主要是`mkfs.c`。
实际上如果你不删除原来的git commit信息，其表现为仅跟踪目录mkfs下的文件`mkfs.c`，其它（新增的）文件是被忽视掉的。
貌似你需要先将`.gitignore`文件中忽视mkfs那一行删除，然后运行`git add ./mkfs/mkfs.c`，运行完之后将还原`.gitignore`文件即可。

## 环境搭建与项目构建

> https://pdos.csail.mit.edu/6.1810/

可去以上MIT的课程网址找到环境搭建的教程。大体上是安装用于交叉编译的gcc和binutils、gdb-multiarch、Qemu。
> xv6-riscv是基于64位RISC-V。

项目的构建工具显然是Make。在安装好上述依赖的软件包之后，使用`make qemu`即可运行项目。

## 代码分析

我们首先打开文件`entry.S`，首先阅读该文件头部的注释可知，每个hart初始都将从地址0x80000000处运行。因此，`kernel.ld`得将代码（实际上是代码汇编得到的节）放到地址0x80000000处。
> hart是RISC-V中的概念，可译为硬件线程。
> `kernel.ld`是针对GCC中链接器ld，称为链接脚本。

接下来在`_entry`中的代码主要是为每个CPU设置内核栈。`la`是Load Address，`li`是Load Immediate，`csrr`是Control and Status Register Read。`mhartid`是RISC-V中CSR的一种，其中保存当前hart的ID，一般从0开始。RISC-V中的sp寄存器中保存栈顶地址，其中sp维护的栈是递减的。最后会通过`call`指令调用`start.c`中的`start`函数。
定位到`start`函数，首先进行的是通过设置`mstatus`寄存器，将RISC-V的特权级模式从M到S。
> RISC-V中存在一个特权级模式，也可以叫做特权等级的概念。目前有三种U、S、M，分别对应用户程序模式、监督者模式、机器模式。

其中使用到的函数`r_mstatus`和`w_mstatus`分别是读`mstatus`寄存器和写`mstatus`寄存器，这样的函数命名是简单明了，在xv6-riscv中，这样的函数还有很多，比如接下来的`w_mepc`和`w_satp`等等，之后便不一一解释了。


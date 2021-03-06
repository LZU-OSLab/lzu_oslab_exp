# Lab4

本实验所需源码可从 [lzu_oslab_exp 仓库](https://git.neko.ooo/LZU-OSLab/lzu_oslab_exp) 下载。

本实验的 step by step 文档可以访问 [Code an OS Project](https://lzu-oslab.github.io/step_by_step_doc/)。

这一节我们将开启分页，从物理地址过渡到虚拟地址，并在此基础上完成虚拟页共享、写时复制等虚拟内存机制。

## 什么是虚拟内存

在启用分页机制前，因为没有使用到页表项中的访问权限控制，内存中的代码可以不受限制地运行在 CPU 上，可以直接访问任何一处物理地址。对于恶意程序，它可以杀死其它程序甚至可以杀死内核，将整个系统的资源据为己有；对于正常程序，它必须小心控制内存访问，避免修改其他程序。显然，这种无限制的环境既给了程序自由，又让程序处于随时可能被破坏的境地。

若程序跑在虚拟内存上，不直接访问物理内存，并且每个程序（进程）都有相同的地址空间。应用程序不再需要担心会被随意破坏或者破坏别的进程，也不再需要关心会被加载到内存中的哪个位置。

尽管程序跑在虚拟内存上，但是最终一定是访问到物理内存，虚拟内存只是操作系统的抽象。每当程序访问某个虚拟地址，硬件和操作系统会将虚拟地址转化为某个物理地址，然后程序访问到对应的物理内存。显然，实现虚拟内存的关键就在于**设置虚拟地址到物理地址的映射**。

考虑这样的情况，程序 A 访问虚拟地址 100，程序 B 也访问虚拟地址 100。对于程序 A，操作系统将虚拟地址 100 映射到物理地址 1000；对于程序 B，操作系统将物理地址映射到物理地址 2000。虽然两程序访问同一地址，但最终却访问到物理内存的不同位置。按照这种思路，让不同的程序拥有相同的虚拟地址空间，将其中的虚拟地址映射到不同的物理地址，这样就实现了“每个进程都有相同的地址空间”的抽象。相同虚拟地址可以映射到不同的物理地址，同样的，不同的虚拟地址也可以映射到同一物理地址，这样就实现了不同程序共享同一块内存。

虚拟内存需要软硬件协作才能够高效实现，并且硬件提供的机制制约着操作系统的实现方式。

## 实现虚拟内存的硬件基础

一个相对完整的 RISC-V 处理器通常会带有一个 MMU（*Memory Manegement Unit*，内存管理单元）来处理和内存管理相关的事务，虚拟地址到物理地址的转换就由 MMU 实现。

内存地址的虚实转换依靠 MMU 读取操作系统提供的页表来完成。硬件完成查询页表，将虚拟地址转换为物理地址的过程，此过程对软件是透明的。但硬件不能直接知道虚拟地址到物理地址的映射，软件（OS）主要的职责就是维护这一映射关系，并在恰当的时候将它写入到页表中。单级页表是一个以页表项为元素的数组，页表项中除了存有对应虚拟地址所指向的物理地址外，还存有读写权限等附加信息。

现代处理器提供的分页机制基本都使用多级页表，将虚拟地址划分多个虚拟页号（*VPN*, *Virtual Page Numer*）和页内偏移（*offset*），RISC-V 的页内偏移均使用 12 位二进制表示（这意味着每一页都是 4KiB）。上一级页表中的页表项（*PTE*， *Page Table Entry*）指向下一级页表的起始物理地址，最后一级页表中页表项指向虚拟地址对应的物理地址。将页表看作页表项数组 `struct pte page_table[]`，某一级 VPN 就是该级页表项数组的索引。以下是查询页表获取物理地址的步骤：

1. 找到一级页表（页目录）的物理地址，这个地址存放在**控制寄存器 satp** 中
2. 根据 VPN 找到包含下级页表起始物理地址的页表项
3. 重复步骤 2，直到查找到最后一级页表
4. 读取页表项中的物理地址

这个过程可以表示为下图（此图为 Sv32 分页方案的页表访问过程，Sv39 与此类似，页表再多一级）：

![virtual address to physical address](images/virtual-to-physical.png)

其中，页表中的叶节点指示虚地址是否已经被映射到了真正的物理页面，如果是，则指示了哪些权限模式和通过哪种类型的访问可以操作这个页。访问未被映射的页或访问权限不足会导致页错误异常（page fault exception）。

在我们操作系统的分页管理中，每个进程都有自己独立的虚拟地址空间，为了避免进程切换后通过 TLB 访问到上一进程的映射，将虚拟地址转换为错误的物理地址，通常发生页表切换后要立刻使用指令 sfence.vma 刷新 TLB，本文中已将此指令以 C 语言内联汇编的形式用 invalidate() 宏将其封装。

> **小知识：TLB**  
> 
> 每次访问内存都要查找页表，查找页表要涉及多次内存访问操作，这会严重影响 CPU 读取指令和读写数据的性能。为了解决这个问题，现代处理器使用了 *TLB*（*Translation-Lookaside Buffer*，转译后备缓冲器）来加快虚拟地址到物理地址的转换。TLB 是缓存虚拟地址到物理地址映射关系的*全相连高速缓存*，可以看作一个数组，元素存储映射关系。访问内存时，先从 TLB 中查找物理地址，如果查找到（TLB hit），就不需要查找页表，避免的耗时的内存访问；如果没查找到（TLB miss），再查找页表并将映射缓存进 TLB。良好的程序都会重复利用*局部性原理*，并且现代处理器还将指令和数据划分开，分别使用独立的 TLB，取指令用 iTLB，读写数据用 dTLB，还可能存在共用的 uTLB，显著提高地址转换的性能。  
> 本内核不对 TLB 进行细粒度的管理，不介绍 TLB 词条的结构。这里给出一个 TLB 的实例，用 perf 分析进程 5200（firefox 浏览器）在 21 秒内的 TLB miss rate：
>
> ```
> SHELL> perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses -p 5200
> 
> ^C
>  Performance counter stats for process id '5200':
>
>     1,760,539,473      dTLB-loads:u
>         6,163,558      dTLB-load-misses:u        #    0.35% of all dTLB cache accesses
>         9,156,828      iTLB-loads:u
>         4,551,108      iTLB-load-misses:u        #   49.70% of all iTLB cache accesses
>
>      21.205687845 seconds time elapsed
>```
>
> 可见，数据 TLB 的命中率接近 100%，指令 TLB 的命中率比较低，但也接近 50%。

### Sv39

RISC-V 标准中给出了若干虚拟内存分页方案，以 SvX 的模式命名，其中 X 是以位为单位的虚拟地址的长度，RV32 仅支持 Sv32，RV64 目前仅支持 Sv39 和 Sv48。其中最受欢迎的是 Sv39, QEMU 和阿里平头哥的玄铁 RISC-V 芯片都支持这种分页方案，因此本系统也采用这种分页方案。平头哥的玄铁 RISC-V 芯片在官方标准的基础上做了些许修改，将不常用到的字段挪作它用，而 QEMU 中 virt 平台则与官方标准相同。

存放一级页表（页目录）的物理地址的控制寄存器 satp 的格式如下图所示。

![satp](images/satp.png)

MODE 表示分页模式，有以下选项。0 表示不采用分页管理（通常 M 模式的程序在第一次进入 S 模式之前会写 0 以禁用分页），没有地址翻译与保护，8 表示采用 Sv39 分页方案。

![satp-mode](images/satp-mode.png)

*ASID*（*Address Space IDentifier*) 用来标识 TLB 词条中映射归属的地址空间，ASID 类似于进程 ID，每个进程都有一个 ASID。TLB 中多个进程的虚拟地址-物理地址映射共存，MMU 在查询 TLB 时会忽略不属于当前进程的映射，在进程切换时只需要修改 ASID，不需要刷新 TLB，降低了进程切换的成本。简单起见我们暂不使用 ASID，选择每次切换进程刷新 TLB。

PPN 表示页目录项的物理页号（物理地址右移 12 位）。

WARL 是 *Write Any values, Read Legal Values* 的简称，表示对应字段可以写入任意值，处理器仅在该字段的值合法时读取，否则忽略该字段。

虚拟地址构成如下：

![virtual address](images/virtual-address.png)

物理地址构成如下：

![physical address](images/physical-address.png)

Sv39 支持 39 位虚拟地址空间，56 位物理地址空间（是的，物理地址空间往往大于虚拟地址空间，32 位机只支持 32 位地址空间指的是虚拟地址空间），最多三级页表，虚拟页大小和物理页相同，在三级页表下页内偏移占据虚拟地址低 12 位，每个页的大小为 $2^{12}B=4KiB$，共$2^{27}$页。为了确保每个页表大小相同，剩下的 27 位将被三等分，因此每个 VPN 9 位，一个页表共有 $2^{9}=512$ 项，每个页表项（PTE）占用 64 bit，每个页表占据一页。

页表项包含下一级页表起始地址或虚拟页对应的物理页的起始地址，但不会直接将完整物理地址写进去，这样占用空间太大了，通常要求页表/页的起始物理地址 4K 对齐（即起始地址的低 12 位全为 0），省去页内偏移，用剩余的空间存放标志位。Sv39 页表项结构如下：

![PTE](images/pte.png)

| 标志位 | 意义 |
| :- | :- |
| RSW | 供操作系统使用，硬件会忽略 |
| D(Dirty) | 自上次清除 D 位以来页面是否被弄脏（如被写入） |
| A(Access) | 自上次清除 A 位以来页面是否被访问 |
| G(Global) | 是否对所有虚拟地址均有效，通常用以标记内核的页表，本实验不使用 |
| U(User) | 是否是用户所有的页。U = 1 则只有 U-mode 可访问，而 **S 模式不能**，否则只有 S-mode 可访问 |
| X(eXecutable) | 结合 U 位判断当前状态是否可执行 |
| W(Writable) | 结合 U 位判断当前状态是否可写 |
| R(Readable) | 结合 U 位判断当前状态是否可读 |
| V(valid) | 有效位。若 V = 0，表示该页不在物理内存中，任何遍历到此页表项的虚址转换操作都会导致页错误 |

从上面的表可以看到，RISC-V 的权限控制非常严格，U-mode 只能访问“用户态”（U 位置位）的页，S-mode 只能访问“内核态”的页。有时确实需要在内核态读取用户态进程所有的页，RISC-V 虽然默认不允许，但仍然为操作系统开了后门。当置位 `status` 寄存器中的 SUM 位时，可以在 S-mode 访问“用户态”页。

此外如果 R、W 和 X 位都是 0，那么这个页表项是**指向下一级页表的指针**，否则它是页表树的一个叶节点。此时它们可以有个特殊用法——实现**大页模式**。当中间某级页表 RWX 位不均为 0 时，CPU 将直接提取出 PTE 中的物理地址，不再考虑下级页表。利用这个特性，将一级页表中的 PTE RWX 某一位置位，一个 PTE 就可以映射 1GiB 地址空间，将二级页表中的 PTE RWX 某一位置位，一个 PTE 就可以映射 2MiB 地址空间。

Clock 算法等一些页面置换算法会使用 A 位和 D 位来判断页面的使用情况。A 位和 D 位的置位有两种模式，一种是 QEMU 所使用的 MMU 自动置位，另一种是平头哥芯片所采用的产生页错误的方案：若 A 位为 0 时进行内存访问，或 D 位为 0 时进行内存写入，都会产生页错误。后者可以通过在操作系统的异常处理程序中手动将其置位达到与前者同样的效果。

## 处理器地址、可执行文件与虚拟内存

程序经过编译链接生成可执行文件，执行时加载进内存，执行指令、访问数据都需要 CPU 访问内存地址，这个内存地址就**硬编码**在可执行文件中。

为了理解可执行文件中的地址和处理器执行时访问的内存地址的关系，在 x86-64 GNU/Linux 中做一个小实验。示例代码如下：

```c
// 文件名：global.c
// 编译命令：gcc global.c
#include <stdio.h>
int global = 1;
int main()
{
    printf("%p\n", &global);
    return 0;
}
```

这个程序经过编译生成汇编代码，汇编后生成 .o 文件（可重定位目标文件），链接后生成 ELF 可执行文件 a.out。生成的 .o 文件未经链接，其中的指令地址都是相对于文件起始地址 0 的偏移，访问地址的指令中地址部分都被编码为 0。在链接阶段，多个 .o 文件聚合为一个可执行文件，指令地址被修改为运行时的**绝对地址**，数据、函数的地址有多种方式表示，但不管使用哪种方法，都一定会获取到一个固定的运行时地址。

运行时，操作系统的加载器会分析可执行文件的格式，查找到程序入口点（C/C++ 程序一般是`main()`），并跳转执行。`main()`函数的地址就是通过解析 ELF 可执行文件获取到的。我们可以使用 readelf ELF 格式可执行文件的完整信息。

查看 a.out 中`global`的地址：

```shell
SHELL> readelf -s a.out | grep global
82: 0000000000404024     4 OBJECT  GLOBAL DEFAULT   23 global
```

可以看到`global`是一个全局符号，大小是 4 字节，类型是 OBJECT（变量），可见性为 DEFAULT（可以理解为具有外部链接），地址是 `0000000000404024`。我们运行该程序查看运行时 `global` 的地址是否和 readelf 看到的一样：

```shell
SHELL> ./a.out
0x404024
```

可见全局符号（非 static 全局变量，非 static 函数）的地址在编译时就已经确定，我们还可以用 `objdump` 反汇编看到每一条指令运行时地址。由于所有指令、全局数据的地址在编译链接时已经确定，我们必须在编译链接时就确定进程的虚拟地址空间布局。

在建立页表之前，首先要规划虚拟内存布局，决定存放内核的虚拟内存空间。考虑这样一个前提：虚拟内存的作用之一就是让进程认为自己独占了整了内存，除此之外为了便于调用系统函数，内核部分也需要在虚拟内存地址空间中有所体现，因此虚拟内存的布局中只有系统内核和当前进程的代码与数据。

本操作系统中，使用 RV32 的进程虚拟地址空间布局，布局如下：

```
0xFFFFFFFF----->+--------------+
                |              |
                |              |
                |    Kernel    |
                |              |
                |              |
0xC0000000----->---------------+
                |    Hole      |
0xBFFFFFF0----->---------------+
                |    Stack     |
                |      +       |
                |      |       |
                |      v       |
                |              |
                |              |
                |      ^       |
                |      |       |
                |      +       |
                | Dynamic data |
       brk----->+--------------+
                |              |
                | Static data  |
                |              |
                +--------------+
                |              |
                |     Text     |
                |              |
0x00010000----->---------------+
                |   Reserved   |
0x00000000----->+--------------+
```

栈从 0xBFFFFFF0 向低地址扩张，因随后即将采用页面大小为 1 GiB 的大页模式，而 0xC0000000 恰好是栈空间往上最近的与 1 GiB 对齐的起始地址，留给内核的空间较大。

内核占据虚拟地址空间的`0xC0000000`到`0xFFFFFFFF`，对应物理内存空间是`0x80000000`到`0x88000000`，内核的起始物理地址和虚拟物理地址恰好相差`0x40000000`，得到虚拟地址和物理地址之间的转换关系：

$$VirtualAddress = PhysicalAddress + LinearOffset$$

内核虚拟地址空间就是可用物理内存地址空间的线性映射：

$$
VirtualAddressSpace[0xC0000000,C8000000) \longrightarrow PhysicalAddressSpace[0x80000000, 88000000)
$$

程序中使用以下宏表示内核虚拟地址空间的线性映射：

```c
// include/mm.h
#define LINEAR_OFFSET    0x40000000
#define PHYSICAL(vaddr)  (vaddr - LINEAR_OFFSET)
#define VIRTUAL(paddr)   (paddr + LINEAR_OFFSET)
```

最开始，内核程序入口点在 0x80200000，转换为虚拟地址后变成 0xC0200000。我们在链接脚本中写入这个地址：

```linker.ld
/* 目标架构 */
OUTPUT_ARCH(riscv)
/* 执行入口 */
ENTRY(_start)
/* 起始地址 */
BASE_ADDRESS = 0xc0200000;
```

这样，可执行文件中的地址全部大于 0xC0200000，我们确保了可执行文件中的地址与规定的虚拟地址空间一致。

## 开启分页

内存管理的一切少不了操作系统的参与，其中最重要的任务之一便是维护每个进程的页表。在我们创建用户进程之前，只有从开机便一直持续运行的内核代码，因此我们的总体思路是先为内核区做一个页表，随后将一级页表的物理地址和分页方案写入 satp 寄存器，最后刷新 TLB，虚拟分页就可成功开启。

本系统内核的代码在编译完成后已大于 4KiB（一页），若手动编写内核页表，既费时且没有灵活性，每次内核大小的增长都将导致页表需要重新编写。因此这里采用更为灵活简便的页表生成方式：进入内核后通过内核生成 Sv39 的三级页表。

这有两种实现的方式：

1. 先使用 1GiB 的大页模式来映射整个物理内存到虚拟内存，随后开启分页，依靠系统代码逐一生成小页表后再重新设置 satp 寄存器，刷新 TLB。
2. 进入系统后在使用物理地址时生成小页表，将一级页表写入 satp 寄存器，刷新 TLB。

本系统采用第一种方案，更为灵活简单。开启分页只需要将页表地址和分页方案写入 satp 寄存器即可。在此之前，需要先构造一个页表。

页表项中每一项需要填写的内容见 Sv39 页表项结构。组合拼接后用算术表示即为 $((0x80000000 >> 30) << 28) | 0x0F = (0x80000000 >> 2) | 0x0F$。

硬编码页表只需要写两项 PTE 即可，其他项均设置为 0。

```assembly
boot_pg_dir:
    .zero 2 * 8
    .quad (0x80000000 >> 2) | 0x0F
    .quad (0x80000000 >> 2) | 0x0F
    .zero 508 * 8
```

构建好页表之后，需要把页表物理地址写入到 satp 寄存器中。根据 satp 寄存器结构，低 44 位是一级页表的物理页号（物理地址 << 12），中间 16 位 ASID 暂不使用，高 4 位是分页方案（Sv39 对应 8），satp 寄存器需要写入 $boot\_pg\_dir << 12 | 8 << (44 + 16)$，最后刷新 TLB，跳转到 main 函数即可。

```assembly
# init/entry.s
    .globl boot_stack, boot_stack_top, _start, boot_pg_dir
    .section .text.entry
_start:
    la t0, boot_pg_dir
    srli t0, t0, 12
    li t1, (8 << 60)
    or t0, t0, t1
    csrw satp, t0
    sfence.vma

    li t1, 0x40000000
    la sp, boot_stack_top
    add sp, sp, t1
    la t0, main
    add t0, t0, t1
    jr t0

    .section .bss
boot_stack:
    .space 4096 * 8
boot_stack_top:
```

前面提到，全局变量、函数的地址在编译时已经确定，我们这里取到的地址的值应该与 readelf 读取到的地址完全一样，但实际上并非如此。符号的地址是固定的，取地址的方式却有多种，取决于编译使用的*代码模型*和编译选项。

假设程序中有一个变量`global`，地址是 1000，编译器生成访问`global`的代码时有多种选择：

- 硬编码地址：直接将地址 1000 写到程序中
- PC 相对寻址：将`global`地址到访问它的指令地址的差值 Offset 写入程序中，通过 PC + Offset 获取地址 1000,
- 位置无关代码：将`global`的地址存放到某个固定的地址中，访问时加载器查找`global`的地址并将其写入，程序从这个地址取出（通常是 PC 相对寻址）`global`的地址，再读写`global`。

位置无关代码一般用于*动态链接*的可执行文件，某些 Linux 发行版上的 GCC 在编译时加入了特定的参数，总是生成*位置无关可执行文件*，为了确保所有的地址都通过 PC-relative 寻址获取。本内核使用 medany 代码模型，并使用 -fno-pie 禁止生成位置无关可执行文件。

从 PC 相对寻址的原理可以看出，程序是否能够获取到正确的地址，完全取决于运行时 PC 的值是否为编译器期待的指令地址。在本实验中，可执行文件中指令地址都从 0xC0200000 开始，而加电后 PC 却指向 0x8020000，可执行文件中指令的地址总是跟 PC 相差 0x40000000。考虑以下这种情况：`global`地址为 0xC0201010，可执行文件指令地址为 0xC0200010，编译器认为执行到这条指令时，PC 的值是 0xC0200010，通过 PC + 0x1000 获取`global`的值，运行时 PC 的值却是 0x80200010，PC + 0x1000 获取到的地址是 0x80201010，跟`global`的值相差 0x40000000。

`la` 即 *Load Address* 是一条汇编指令，用于取符号的地址，在这里被拓展为 auipc/addi 指令序列（PC 相对寻址），获取虚拟地址需要加上 0x40000000。

为方便起见，创建一个全局变量 `pg_dir` 指向当前页目录（一级页表），所有和页表有关的函数，都通过 `pg_dir` 来处理页表，初始值为 `entry.s` 中的 `boot_pg_dir`。

完成以上工作后，我们开始创建内核的线性映射。在进入内核后，申请一页空闲物理页并作为新的页目录。

```c
uint64_t page = get_free_page();
assert(page, "mem_init(): fail to allocate page");
pg_dir = (uint64_t *)VIRTUAL(page);
```

创建映射的过程就是遍历页表，若页表不存在就创建，直到最后一级页表，将物理页号和标志位写进 PTE。

有了`pug_page()`，只需要一页一页地映射虚拟页和物理页即可。

```c
void map_kernel()
{
    map_pages(MEM_START, MEM_END, KERNEL_ADDRESS, KERN_RWX | PAGE_VALID);
    map_pages(PLIC_START, PLIC_END, PLIC_START_ADDR, KERN_RW | PAGE_VALID);
    map_pages(UART_START, CEIL(UART_END), UART_START_ADDR, KERN_RW | PAGE_VALID);
}
```

内核区仅仅是物理内存的线性映射，内核要通过内核区的线性映射无限制地访问物理内存，所以为内核区的页开启了 RWX 权限。

创建这个映射的过程中没有修改系统正在使用的页表，还需要切换页表，让新映射生效。

```c
/**
 * @brief 激活当前进程页表
 * @note 置位 status 寄存器 SUM 标志位，允许内核读写用户态内存
 */
void active_mapping()
{
    __asm__ __volatile__("csrs sstatus, %0\n\t"
                 "csrw satp, %1\n\t"
                 "sfence.vma\n\t"
                 : /* empty output list */
                 : "r"(1 << 18),
                   "r"((PHYSICAL((uint64_t)pg_dir) >> 12) |
                   ((uint64_t)8 << 60)));
}
```

我们给了内核读写用户态内存的特权，这样做可能会导致一定的安全问题，但方便了应用态和内核态之间的数据传送。

至此，我们从物理地址过渡到了虚拟地址。

## 虚拟内存页的分配与释放

与物理内存相对应的，虚拟内存也可以按页申请与释放。在虚拟内存页的分配与释放过程中，除对物理内存页进行操作外，最重要的工作是对页表与页表项的修改。虚拟内存页的释放并无具体的使用场景，通常是以给定页表项释放其物理页并使页表项无效的形式出现（如页面换出），不做单独考虑。

在 get_free_page() 函数与 put_page() 函数配合之下，给定某一虚拟地址与页表项标志位，实现分配对应虚拟内存页函数较为简单：

1. 使用 get_free_page() 函数获取一页新物理页，得到其起始地址 paddr；
2. 使用 put_page() 函数将给定虚拟地址 vaddr 与标志位 flag 映射到物理地址 paddr 之上。

## 写时复制

系统中往往有多个进程共存，每个进程（除了进程 0）都有父进程，子进程由 `fork()` 函数创建，会继承父进程的代码、数据，但又有自己不同的代码、数据。

```c
#include <stdio.h>
#include <unistd.h>
int value = 100;
int main()
{
    pid_t pid = fork();
    if (pid < 0) { // fork() 失败
        fprintf(stderr, "fork() failed\n");
    } else if (pid == 0) { // 父进程
        value = 200;
        printf("parent's value: %d\n", value);
    } else { // 子进程
        printf("child's value: %d\n", value);
    }
    return 0;
}
// 可能的输出
// child's value: 100
// parent's value: 200
```

以上代码片段创建一个子进程，并分别在父子进程中打印 `value` 的值。在父进程中，`value` 的值是 `100`，`fork()` 出子进程后，子进程也拥有一份 `value` 的副本，两个进程的值互不干扰。在这个简单的例子中，父子进程共享代码并拥有不同的 `value`（`value`的不同副本），在更复杂的多进程环境下，多个进程可能拥有更多的变量。

实现这种“继承”的最简单办法是直接拷贝。以上面的程序为例，在 `fork()` 子进程时，可以把父进程的全部代码、数据拷贝一份给子进程。然而，父子进程的代码一定是相同的，只有数据可能会被修改，直接拷贝将进行大量不必要的拷贝。解决直接拷贝的低效问题的方法是写时复制。写时复制共享代码、数据，将拷贝延迟到进程修改数据时。

```
              ┌─────────────────┐                                     ┌─────────────────┐
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │◄─────────Parent Process
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │
              │    Code/Data    │◄─────────Parent Process             │    Code/Data    │
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │◄─────────Child Process
              │                 │                                     │                 │
              │                 │                                     │                 │
              │                 │                                     │                 │
              └─────────────────┘                                     └─────────────────┘



──────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────
                                                      │
                                                      │
                                                      │
                                                      │
                                                      │  Child modify data
                                                      │
                                                      │
                                                      │
                                                      │
                                                      ▼



               ┌─────────────────┐                Copy                 ┌─────────────────┐
               │                 │                                     │                 │
               │                 │    ─────────────────────────────►   │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │    Code/Data    │◄─────────Parent Process             │    Code/Data    │◄─────────Child Process
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               │                 │                                     │                 │
               └─────────────────┘                                     └─────────────────┘
```

在上面的图示中，父进程创建子进程后，两个进程共享同一块代码、数据，暂不拷贝。子进程修改数据时，内核仅将**这块数据**拷贝一份给子进程，这时父子进程不再共享这块数据，子进程在它自己的副本上修改。

为了实现写时复制，内核必须发现并拦截进程的写操作，在进程写之前拷贝数据。回忆前面的页表结构，页表项中有标志位 W 控制物理页是否可写，程序试图写只读的物理页将产生异常（写保护）。不同进程共享同一物理页时，我们清空标志位 W，禁止进程写该页，这样在进程写该页时，处理器产生异常，内核将发现并捕获异常，从而拦截进程的写操作。在内核捕获异常后，再拷贝数据并设置标志位 W。

### 共享（拷贝）虚拟页

了解写时复制的原理后，现在来实现虚拟页的共享。共享虚拟页其实就是让两个进程的虚拟页映射到同一块物理页。

```
VP: Virtual Page
PP: Physical Page

  VP1              VP1    VP2
   │                │      │
   │                │      │
   ▼                ▼      ▼
┌──────┐   Share    ┌──────┐
│  PP  │  ────────► │  PP  │
└──────┘            └──────┘
```

完成共享只需要设置虚拟页到物理页的映射关系。由于是两个进程共享一个物理页，需要同步修改两个进程的页表，而且两个页表都是三级页表，这给实现带来一定难度。

系统仅在创建进程时共享虚拟页，而创建进程不太可能仅共享几页虚拟页，因此我们将共享的最小虚拟页数量设置为 512 个（2 M），这个数量恰好是一个第三级页表所能够容纳的页表项数。也就是说，每次至少共享一个第三级页表管理的虚拟内存区域。

两个进程页表的修改必须保持同步，比如将进程 A 的 [100, 500) 拷贝到进程 B 的 [400, 800)，进程 A 的​ 100 对应进程 B 的 400，进程 A 的 200 对应进程 B 的 500。将多级页表看成一棵高度为 3 的多叉树，节点为页表，将当前进程虚拟地址 [from, from + size) 拷贝到目标进程虚拟地址空间 [to, to + size) 就是层次遍历两颗树，将 [from, from + size)对应的页表项拷贝到 [to, to + size) 对应的页表项中并进行写保护。

遍历的过程中存在边界条件，比如上诉进程 A 地址 [100, 500) 在一个页表中，而进程 B 地址 [400, 800) 在两个页表中。我们要考虑到遍历过程中地址区域在不同页表中的情况。具体实现使用一主一从的方式，源进程的页表树为主，目的进程的页表树为辅，在遍历源页表树的过程中访问目的进程页表树。

为了避免权限问题，两地址区间要么都是用户空间，要么都是内核空间。其中内核态的地址空间是内核，所有进程共享，不需要拷贝，所以不需要写保护，具体代码实现请参考 `mm/memory.c` 中的 `copy_page_tables()` 函数。

### 取消写保护

在拷贝虚拟地址空间时对虚拟页进行写保护，程序尝试写只读虚拟页会产生异常，内核将捕获异常并处理。因此，取消写保护的操作要作为异常处理函数。

写只读内存页对应的异常是 *Store page fault*，取消写保护应作为 *Store page fault* 的异常处理例程。在以后的实验中，我们会遇到其他的 *page fault* 异常，因此我们对所有的 *page fault* 异常通过 `page_fault_handler()` 函数统一处理。

取消写保护时，置位发生异常的地址对应的页表项 W 标志，拷贝物理页。有一种特殊情况，虽然虚拟页标记为只读，但该页没有被共享，这时只需要置位 W 标志即可。

发生 *page fault* 时，发生异常的虚拟地址将被保存在 stval 寄存器中，我们通过虚拟地址定位页表项。

## 动态内存分配与释放

在系统中，除了使用全局变量这种静态的内存分配方式外，往往还需要更灵活地在运行过程中根据需求获得一块指定大小的连续内存空间，用于缓冲区、链表等重要数据结构的存放，这就需要操作系统或支持库实现动态的内存分配。

在 C 语言的标准库中有 malloc/free 函数用于实现动态内存的分配与释放，但本操作系统目前尚未移植标准库，因此需要自己动手编写 malloc/free 函数。

在我们的实现中，桶是专用于存放动态内存分配块的**页**，每一个桶只存放一类大小的块。例如分配一块 1KiB 大小的内存空间，需要从“1KiB 桶”中取得，该桶总共可分配 4 块 1KiB 大小的内存空间。若需要分配一块 2KiB 大小的空间，不能从先前分配的 1KiB 桶中取得，需要重新分配一页物理页充当“2KiB 桶”，该桶可分配 2 块 2KiB 大小的空间。动态内存分配支持分配 $16B \leq 2^nB \leq 4KiB$ 大小的块，暂不支持超过 1 页大小的分配。按桶分配可能产生的内碎片根据如下公式计算最多约为 28KiB，浪费的物理内存空间非常少。

$4KiB \times 8 - 16B-32B-64B-128B-256B-512B-1KiB-2KiB=28688B \approx 28KiB$

对于每一个桶，都需要一个桶描述符用于记录桶的信息，桶描述符使用结构体实现，结构体中的域及说明见下表。桶描述符只记录桶中第一个空闲块的索引，其余空闲块的索引通过模拟静态链表的形式存放于前一空闲块中。空闲块的顺序并非永远从低地址向高地址排序，而是根据释放的顺序排序，最近释放的空闲块以头插法插入静态链表。操作系统中有一全局数组 `struct bucket_desc* bucket_dir[]` 存放了每一类桶描述符的链表。

| 域 | 说明 |
| :- | :- |
|next | 下一个同类桶描述符 |
|page | 桶的起始内存地址 |
|freeidx | 本桶中第一个空闲块的索引 |
|refcnt | 已分配块计数 |

### 创建新桶

创建新桶的步骤如下：

1. 申请一页新的物理页；
2. 使用 init_bucket_page() 初始化桶，在桶中每一块空间写入下一空闲块的索引；
3. 获取空间存放桶描述符；
4. 填写桶描述符；
5. 将桶描述符以头插法插入桶描述符链表 bucket_dir[size]。

由于桶描述符所用的内存也需要使用 malloc 来动态分配，因此创建新桶时需要将桶描述符大小（32B）的桶作为特殊桶与其他类型的桶分类讨论。在建立普通桶时，桶描述符存放在使用 malloc 分配而来的内存块中；建立 32B 桶时，将桶描述符放入新桶的第一块，可以避免循环调用 malloc 分配第一个 32B 桶的问题。填写桶描述符时，普通桶的引用计数为 0，且本桶中第一个空闲块的索引为 0；32B 桶的引用计数为 1，且本桶中第一个空闲块的索引为 1. 因此特殊桶还有一个性质：桶描述符与桶所在内存地址一致，通过这一特性可以很方便判断当前桶是否为特殊桶，这将会在内存块释放中被使用。

### 分配内存块

在使用 kmalloc_i() 函数分配内存块时所需内存空间不一定恰好满足$16B \leq 2^nB \leq 4KiB$大小。若请求的空间超过 4KiB，则申请失败；请求空间小于 16B，可直接分配 16B
块；请求空间不在这一离散范围内，需要计算满足条件的最小块。

随后从对应大小的桶链表 bucket_dir[n-4] 中查找有空闲块的桶，若没有合适的桶则使用 take_empty_bucket() 创建一个新桶。最后修改桶描述符中的引用计数与第一空闲块信息，返回新获得的块的起始内存地址。

### 释放内存块

使用 kfree_s_i() 函数释放内存块同样需要知道内存块大小，若已知需要释放页的大小，则可按照分配时的计算方式计算，但通常上层函数接口不会请求需要释放的大小，只提供需要释放的内存块起始地址，这时候除了枚举所有桶外，还可以主动推测块大小的范围。

q_pow2_factor() 函数使用德布鲁因序列快速计算素因子中所含 2 的个数，地址的因数中有 n 个 2，则根据内存块分配的对齐要求可知这一内存块最大不超过$2^nB$。

找到桶的范围后还可以通过将内存块的地址做向下的 4KiB 对齐计算得到内存块对应桶的起始地址，可以很方便地通过匹配桶描述符中的 page 域检查内存块是否属于某一桶。

从最小的桶开始查找到最大可能尺寸的桶，若没有找到，则表明欲释放的内存块起始地址有误。

若找到了需要释放的块所在桶，需要将桶描述符的引用计数 refcnt 加 1，随后通过引用计数判断该桶是否已空：引用计数为 1 且桶描述符与桶所在内存地址一致，表明这是一个特殊空桶（32B桶），对于普通桶，引用计数为 0 为空。

若引用计数表明该桶未空，修改第一空闲块的索引 freeidx 为该块，并在该块内填写原第一空闲块的索引。若引用计数表明该桶已空，则需要释放这一空桶的物理页，并将桶描述符从链表中摘除，如果该桶不是特殊桶（桶描述符与桶所在内存地址一致），还需用 kfree_s_i() 释放桶描述符的内存块。

释放内存块结束后返回释放的内存块大小，以满足 POSIX 标准的要求。

## 实验内容

1. 请将你在 Lab3 中填充的代码复制到 Lab4 中：
    - mm/memory.c 中的 `mem_init`, `free_page`, `get_free_page` 函数
    - lib/string.c 中的 `memset` 函数
    - 请注意：与 lab3 不同，lab4 中的 init/entry.s 无需合并
2. **根据上述解析、参考文档、代码注释与关联内容代码**，补全以下文件中的代码，使得系统启动后运行的物理内存测试可正常通过：
    1. mm/memory.c 中的 `put_page()`, `get_pte()` 与 `un_wp_page()` 函数
    2. mm/malloc.c 中的 `take_empty_bucket()` 函数
    3. 推荐顺序：`put_page()`, `get_pte()`, `un_wp_page()`, `take_empty_bucket()`.
    4. 请注意：代码编写过程中注意物理地址与虚拟地址的转换，开启分页后所有访问的地址都是虚拟地址，获取到的物理地址可以使用 VIRTUAL() 宏转换为内核段线性映射的虚拟地址。
3. 回答以下问题：
   1. RISC-V 提供的虚拟地址空间是连续的吗？如果不是，请指出合法的虚拟地址空间？
   2. RISC-V 提供的虚拟地址空间和进程的虚拟地址空间是一个东西吗？
   3. 请写出临时页表 `boot_pg_dir` 中的两个有效映射，并解释创建这两个映射的原因。
   4. 为什么要重新建立页目录和页表，而不直接修改原来的页表？
   5. 为什么链接后的内核中所有的地址都是 `0xc0200000` 之后的，但 sbi 启动到 `0x80200000` 还可以正确执行？
   6. 在本章实验代码中，系统在何时创建虚拟地址到物理地址的映射？
   7. 何时会出现虚拟页只读，但未共享的情况？
4. 拓展任务（本次拓展任务 2、3 难度较大）：
    1. 删去临时页表 `boot_pg_dir` 中的第一项有效映射（实际上是错误的做法），在 QEMU 中执行似乎没有问题，请说说你对可能的原因的猜测，有能力的同学可以尝试验证你的猜测。
    2. 本章实验内容有何缺陷与可改进之处？

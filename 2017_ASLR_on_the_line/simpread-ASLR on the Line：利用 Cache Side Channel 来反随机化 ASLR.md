> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 http://www.freebuf.com/sectool/165445.html

*** 本文原创作者：bin2415，本文属 FreeBuf 原创奖励计划，未经许可禁止转载**

## 背景介绍

**该文要介绍的工作是利用 cache 的 side channel 来实现 ASLR 的 derandomization（即获得在 ASLR 下的内存地址）。参考的是发表在 NDSS17 年上的论文 [ASLR on the Line: Practical Cache Attacks on the MMU](https://www.cs.vu.nl/~giuffrida/papers/anc-ndss-2017.pdf)，下面首先介绍下背景知识**

### ASLR

ASLR 是防御与内存有关安全漏洞的一个非常有用的措施，由于其 overhead 比较低，因此被广泛部署在现在的计算机系统中。如图 1 所示，ASLR 是在程序运行时选择将数据和代码放到一个随机的位置，在 64 位机器上，其有效的虚拟地址位数一般为 48 位 (256TB)，地址的熵比较大，可以有效的防止暴力破解 ASLR。

[![](http://image.3001.net/images/20180315/15211008015803.png!small)](http://image.3001.net/images/20180315/15211008015803.png) 
图 1 虚拟地址空间

![pic1_aslr.png](http://image.3001.net/images/20180315/15211008015803.png!small) 
图 1 虚拟地址空间

由于 ASLR 的部署，对程序的 code reuse 攻击 (ret-to-libc, rop 等) 就变的比较困难。所以一般在进行 code reuse 攻击之前，会通过一定的手段 (内存泄露，buffer over read) 来泄露数据段或代码段的地址，从而有效的实施攻击。但是如果没有 buffer over read 等漏洞，则很难泄露相关段的地址，从而很难破解 ASLR 防御。此文 [1] 利用 cache 的 side channel 来实现 ASLR 的 derandomization，此攻击利用了硬件特性来对 ASLR 进行破解，因此不需要程序有 buffer over read 等漏洞。

### Memory Organization

图 2 是 Intel 处理器的内存组织结构，当执行单元（取指令）或者 Load/Store 单元（访问内存）时，其访问到的地址是虚拟地址，需要将虚拟地址转换成物理地址，在计算机系统中，将虚拟地址转换为物理地址的单元为 MMU，其首先检查 TLB(Translation Lookaside Buffer) 表中是否存在需要查找的 Virt Addr, 如果 TLB 中不存在 Virt Addr 到其对应的物理地址的映射，则需要进行 PT Walk，具体的 PT Walk 会在下面具体介绍，此时只需要知道其需要查找 Page Table 表，而 Page Table 表的内容则可作为数据可能会存储在 cache 中。当 Page Table Walk 之后，就找到了 Virt Addr 对应的物理地址 Phys Addr。首先查找 cache，看 cache 中是否存在 phys addr 地址对应的内容，如果 cache 中不存在，则从 DRAM 内存中取得数据。

[![](http://image.3001.net/images/20180315/15211008376214.png!small)](http://image.3001.net/images/20180315/15211008376214.png) 
图 2 Memory orgainzation in a recent Intel processor[1]

![pic2_memory_organization.png](http://image.3001.net/images/20180315/15211008376214.png!small) 
图 2 Memory orgainzation in a recent Intel processor[1]

需要注意的是在现在的 Intel 处理器中（Skylake i7-6700K），cache 是分为三层的，每一层都比其下一层小，但是访问速度较快，相当于每一层都是下一层 cache 的缓存。区中，在第一层 L1 中，又具体将数据 Data 和代码 Code 进行分开，分为 L1 code 和 L1 data；在第二层 L2 和第三层 L3 中 code 和 data 是放在一起的。需要注意的是，L2 中的内容不包含 L1 中的内容，即如果一个地址的数据或代码存放在 L1 中，其肯定不会存放在 L2 中，而 L3 中的内容即包含 L1 中的内容又包含 L2 中的内容，即如果一个地址的数据或代码在 L1 或者 L2 中，其一定存放在 L3 中，相反，如果一个地址的内容不在 L3 中，其一定也不在 L1 和 L2 中。通过此特性，可以使某个内容不在 cache L3 中，来确保其不在 cache 中，从而实现 cache 的 side channel 攻击。

### Page Table Walk

在计算机系统中，程序所能看到的只能是虚拟地址，而真实存在的数据或者代码是存放在物理地址的，所以当程序访问某一虚拟地址 va 时，需要特定的硬件单元 (MMU) 来进行虚拟地址到物理地址 pa 的转换。该转换在计算机系统中是通过查找 page table 来完成的，该过程叫做 Page Table Walk。

[![](http://image.3001.net/images/20180315/15211008501085.png!small)](http://image.3001.net/images/20180315/15211008501085.png) 
图 3 Page Table Walk

![pic3_ptwalk.png](http://image.3001.net/images/20180315/15211008501085.png!small) 
图 3 Page Table Walk

图 3 是我从深入理解计算机系统 [2] 中截取的 Intel i7 的 page table 转换图，由图 3 可知，其虚拟地址是 48 位，由于每个页表是 4KB(2^12B), 所以低 12 位表示页内偏移。由于是在 64 位机器中，每一个 PTE（page table entry）的大小是 64 位, 即 8B（2^3B），所以可用 9 位（2^12/2^3=2^9）表示页表中的偏移，因此可将虚拟地址划分为 4 级页表。首先通过 CR3 寄存器取出一级页表的基地址，再通过 VPN1 找到二级页表的基地址，如此往复，直到找到物理页的基地址 pba, 是的 pba+vpo 即为要转换到的物理地址 pa。

### Cache

当从虚拟地址 va 转换成物理地址 pa 之后，就可以通过物理地址 pa 来访问真实的数据。由于 CPU 处理速度远远大于 DRAM 的访问速度，如果直接访问 DRAM，则可能造成 CPU 处理能力的浪费，所以在现在的计算机系统中，在 CPU 和 DRAM 之间又添加缓存，用来存储 CPU 最近时间内访问的数据或指令。

[![](http://image.3001.net/images/20180315/15211008618539.png!small)](http://image.3001.net/images/20180315/15211008618539.png) 
图 4 cache 结构 [2]

![pic4_cache.png](http://image.3001.net/images/20180315/15211008618539.png!small) 
图 4 cache 结构 [2]

图 4 是当今计算机系统中的 cache 结构，由图中可知，其将物理地址分成了 3 部分：块内偏移，组号和标签。每一个 cache 组可能由多个 cache 行组成，每个 cache 行所存储的数据大小是 B=2^b 字节，则块内偏移就是低 b 位；cache 中存在 S=2^s 个 cache 组，则组号为中间 s 位，剩余的部分就是标签位。当要查找 cache 数据时，首先通过中间 s 为找到 cache 组号，在通过标签位找到其对应的 cache 行，再通过低 b 位找到对应数据。

## Derandomizing ASLR

由前面一节介绍的基础知识我们可以知道，当 MMU 进行虚拟地址 va 到物理地址 pa 的转换时，首先查找 TLB 表，如果 TLB 表 miss，则进行 page table walk。在进行 page table walk 时，就是进行 4 级页表的访问操作。具体的例子如图 5 所示。

[![](http://image.3001.net/images/20180315/15211008752060.png!small)](http://image.3001.net/images/20180315/15211008752060.png) 
图 5 虚拟地址转换 [1]

![pic5_vatrans.png](http://image.3001.net/images/20180315/15211008752060.png!small) 
图 5 虚拟地址转换 [1]

由图 5 可知，首先访问 level 4 页表，通过 CR3 得到 level 4 页表基址的物理地址 b1，再获得页表项 PTE 200（PTE 200 物理地址 pa1：b1+200<<3）上存的数据，而 PTE 200 上存的数据放在 cache 或者 DRAM 中。如果我们通过某种手段（cache side channel）得知 PTE 200 存放在 cache 中对应的组号和块内偏移，则相当于知道了 pa1 的低 b+s 位，而如果 b+s>9+3，则可以通过 cache side channel 知道 200 这个值。同理可以通过 cache side channel 知道其它三层页表内的偏移值: 300, 400, 500。

[![](http://image.3001.net/images/20180315/15211008853122.png!small)](http://image.3001.net/images/20180315/15211008853122.png) 
图 6 derandomize ASLR

![pic6_derandomize.png](http://image.3001.net/images/20180315/15211008853122.png!small) 
图 6 derandomize ASLR

而在当今的计算机系统中，一般都能保证 b+s>12。在 Intel Skylake 处理器中，其对应的 b=6，s=10。

### 攻击模型

该攻击模型是假设攻击者可以在被攻击者的浏览器中执行 JavaScript 代码（通过引导被攻击者访问恶意的网站或者伪装的网站）。

### JavaScript Timing

如果想要使 cache side channel 实施成功，则需要比较精确的 JavaScript 计时器。通过 JavaScript 计时器可以告诉攻击者其访问的数据是存放在 cache 中还是存放在 DRAM 中，因为访问 cache 的速度远远快于访问 DRAM 的速度。在以往的攻击中，一般通过 JavaScript 的 performance.now() 函数来比较精确的获得访问 DRAM 和 cache 的时间差。而现在的浏览器一般都对 side channel 有了一定的防御措施，即将 performance.now() 函数的精度将为 5us。

[![](http://image.3001.net/images/20180315/15211008978463.png!small)](http://image.3001.net/images/20180315/15211008978463.png) 
图 7 performance.now() 进行 side channel 示意图

![pic7_performance.png](http://image.3001.net/images/20180315/15211008978463.png!small) 
图 7 performance.now() 进行 side channel 示意图

由图 7 可知，当 performance.now() 精度很高时，可以在访问数据前记一下时 t0，访问数据后记一下时 t1，将其 T=t1-t0 就记为访问数据的时间，如果 T 与 CT 比较接近，则证明其访问的是 cache，否则，其访问的是内存。而如果将 performance.now() 的精度降为 5us 之后，就如图 7.2 所示，无法分辨其访问的数据是在 cache 中，还是在内存中。

### Shared Memory Counter

在当今浏览器中，采用了 SharedArrayBuffer 函数来申请一个共享的 Buffer，使多个线程能够共享该 Buffer 的数据。

首先申请共享内存:

```
buf = new SharedArrayBuffer(1)

```

在线程 1 中，进行 counter:

```
c=0;
while(buf[0] == 0);

while(buf[0] == 1){
    c++;
}

```

在线程 2 中，访问内存：

```
buf[0]=1;
operation();
buf[0]=0;

```

这种方法进行的 timing 与 performance.now() 的精度没有任何关系，因为它只依赖于 CPU 的运算速度。

### Triggering MMU Page Table Walks

前面也提到过，如果想要进行 Page Table Walks，则首先保证 TLB 缺失，即要操作的目标虚拟地址 va 不在 TLB 表中。所以需要先设计一个 TLB eviction set，保证 TLB 中不存在目标虚拟地址 va。由于 TLB 分为 i-TLB 和 d-TLB，即数据和代码不是在一个 TLB 中，所以在此文 [1] 中，有两种方式来设计 TLB eviction set。首先如果目标虚拟地址 va 在 heap 上，则通过 ArrayBuffer 类型来作为 TLB-eviction set，由于 ArrayBuffer 是页对其的，因此可以有效的预测 ArrayBuffer 的索引与页内偏移的关系，能够比较有效的设计 TLB eviction set。而如果目标虚拟地址 va 在代码段，则需要生成 JITed 代码区域。

### EVICT+TIME Attack

一般常见的 cache side channel 有 EVICT+TIME[3]，PRIME+PROBE[4]和 FLUSH+RELOAD[5]。以上三种 cache side channel 一般都是在不同的场景下来分辨出某些操作 (AES 加密算法等) 涉及到的某些数据或代码是否在 cache 中，通过此种方法来还原具体的操作过程。此文 [1] 采用了两种方法 PRIME+PROBE 和 FLUSH+RELOAD 来分辨出目标地址在进行 Page Table Walks 的时候四个页表项 (有四级页表，每个页表根据 offset 来得到页表项) 映射到了哪些 cache set 中。通过得到页表项映射的 cache set 编号，也就得到了物理地址的中间 s 位。由于 PRIME+PROBE 需要构造精确的 LLC eviction set，在没有精确的计时器的情况下构造该 set 比较困难，所以在此主要介绍 EVICT+TIME 方法的攻击，如果想要了解 PRIME+PROBE 攻击，请参考[1]。

对于一个页表来说，其大小是 4KB，每个页表项为 8B（64 位），假设 cache line 存储的数据大小是 64B，则每一个 cache line 可以存储 8 个页表项，因此一个页表一共占了 64 个 cache lines。所以可以将整个的 LLC（第三层 cache）按每页进行划分，一共可以划分为 64 种不同的颜色。

[![](http://image.3001.net/images/20180315/15211009283046.png!small)](http://image.3001.net/images/20180315/15211009283046.png) 
图 8 page colors

![pic8_page_colors.png](http://image.3001.net/images/20180315/15211009283046.png!small) 
图 8 page colors

具体攻击过程如下：

> 1): 首先选一些内存页来作为 eviction set。这些 eviction set 必须保证能把某一个特定的 page color 对应的 cache line 都给驱逐走，保证目标页表项不在特定的 page color 对应的 cache line 中。
> 
> 2): 对于目标页表项来说，如果它位于页表的第 t 个 cache 行（64 个中的一个），则通过读取 eviction set 中那些内存中相应的 offset（64 个中的一个）来驱逐页表项。offset 是从 1 到 64 进行遍历的。
> 
> 3): 在此访问目标虚拟内存 va，监控它的访问速度。如果它的访问速度比较慢，则说明 t=offset，即 offset 对应的 cache 行与 t 对应的 cache 行一致。此时，就可得到一个页表项。使得 offset 从 1 遍历到 64，则可得到 4 个页表项对应的 cache offset。

### Sliding PT Entries

此时已经得到了 4 个页表项对应的 cache offset，即得到了四个页表项的物理地址的中间 s 位，然而现在面临着两个问题：

> 无法准确的知道每个页表项分别对应的 cache offset，即无法建立四个页表项与得到的 cache offset 的一一映射关系。
> 
> 无法知道 cache 行的页内偏移，也就无法知道物理地址的低 b 位。

此文 [1] 使用了滑动页表项这一方法有效的解决了此类问题。

[![](http://image.3001.net/images/20180315/15211009443430.png!small)](http://image.3001.net/images/20180315/15211009443430.png) 
图 9 sliding PT Entries

![pic9_sliding_pte.png](http://image.3001.net/images/20180315/15211009443430.png!small) 
图 9 sliding PT Entries

为了更清楚的解释这一方法，我们可以看一下图 9 这个例子。在图 9 中，设其此时的虚拟地址为 v，我如果想要在 Page Table Walks 时访问 Level 1 的第 501 个页表项，则需要访问的虚拟地址为 v+4KB；如果想要访问 Level 2 的第 401 个页表项，则需要访问的虚拟地址为 2MB；如果想要访问 Level 3 的第 301 个页表项，则需要访问的虚拟地址为 v+1GB；如果想要访问 Level 4 的第 200 个页表项，则需要访问的虚拟地址为 v+512GB。

### PTL1 和 PTL2

如果想要知道 PTL1 对应的 cache offset，则将 v+i*4KB(i=1,2,…,8)，如果检测到一个 cache line 在 i 的时候发生了变化，则可以得到两个信息：那个变化的 cache line 就是 PTL1 所在的 cacahe line，因此可以知道 level 1 中的页表项对应的 cache 偏移；同时 cache 的块内偏移是 8-i。因此也就知道了 PTL1 的物理地址的低 (s+b) 位。

同理对于 PTL2 来说，可以将 v+i*2MB(i=1,2,…,8)，从而可以检测出 PTL2 对应的 cache offset 和 cache 的块内偏移。

### PTL3 和 PTL4

由于对于 PTL3 和 PTL4 来说，相邻的页表项需要加的偏移较大（1GB 和 512GB），所以针对不同的浏览器内存分配策略有不同的方式进行处理。

**Firefox**

在 Firefox 中，[1] 发现可以使 JavaScript 分配 TBs 级别的虚拟内存，只要对应的需要内存页不被访问 (not touched)，就不会分配具体的物理内存页，因此，针对 Firefox 来说，可以采用反随机化 PTL1 和 PTL2 的方法来反随机化 PTL3 和 PTL4。

**Chrome**

在 Chrome 浏览器中，当内存分配器使用 mmap 分配内存之后会将该内存初始化，因此分配多少虚拟内存就对应着相同的物理内存大小。因此不可能分配 TBs 级别的虚拟内存。[1] 采取的措施是使 JavaScript 申请 2GB 的虚拟内存，一般 2GB 的虚拟内存对应 3 个 Level 3 的页表项（除一些特殊的情况对应 2 个页表项），因此在这 3 个页表项进行测试。如果 v 对应的 Level 3 的页表项在 cache 中的偏移是 7 或 8，则能够检测出来，否则，检测不出来。

## 实验结果

该文 [1] 通过多个指标来衡量该攻击模型，在此仅从两方面进行分析：成功率和可行性。

### 成功率

图 10 显示的分别是在 Chrome 和 Firefox 下的成功率和假阴性和假阳性的比例。其中假阴性是指当在进行 sliding PT entries 时没有检测到 cache line 发生了变化，假阳性是报告出一个错误的地址（与真实的地址 v 不一致）。

[![](http://image.3001.net/images/20180315/15211009648827.png!small)](http://image.3001.net/images/20180315/15211009648827.png) 
图 10 Success Rate

![picx_success_rate.png](http://image.3001.net/images/20180315/15211009648827.png!small) 
图 10 Success Rate

### 可行性

图 11 显示的是随着时间的增加，剩下的还没有解出的虚拟地址的位数关系。

[![](http://image.3001.net/images/20180315/15211009801544.png!small)](http://image.3001.net/images/20180315/15211009801544.png) 
图 11 Fesibility

![picx1_fesibility.png](http://image.3001.net/images/20180315/15211009801544.png!small) 
图 11 Fesibility

由图 11 可知，其可以在较短的时间内解出大部分的虚拟地址位数。即使没有解出所有的位数，也降低了暴力破解的熵值，极大地提高了暴力破解成功的可能性。

另该项目的具体实现作者 [1] 已经发布在 github [revanc](https://github.com/vusec/revanc) 上，但是其发布的是针对 C 程序 ASLR 的反随机化。

## 缓解措施

最后，此文 [1] 也提供了一些缓解措施：

> 检测：由于实行该攻击需要进行大量的虚拟内存分配和 cache 的清空操作，在一定程度上影响了浏览器的性能，因此可以通过性能检测器来检查此类漏洞。但是此类检测被证明 [1] 存在假阴性和假阳性。
> 
> 安全的计时器: 由于 cache side channel 需要准确的计时器，所以可能通过设计安全的计时器来有效的防止 cache side channel。但是 [6] 也总结了其它有效的方式来实施 cache side channel 攻击
> 
> 隔离的 caches：可以设计一个专门的 cache 来存储页表项的数据。

## 引用

[1] Gras, B., Razavi, K., Bosman, E., Bos, H., & Giuffrida, C. (2017). ASLR on the Line: Practical Cache Attacks on the MMU. NDSS (Feb. 2017).

[2] 深入理解计算机系统 (第三版 英文版)

[3] Tromer, E., Osvik, D. A., & Shamir, A. (2010). Efficient cache attacks on AES, and countermeasures. Journal of Cryptology, 23(1), 37-71.

[4] Liu, F., Yarom, Y., Ge, Q., Heiser, G., & Lee, R. B. (2015, May). Last-level cache side-channel attacks are practical. In Security and Privacy (SP), 2015 IEEE Symposium on (pp. 605-622). IEEE.

[5] Yarom, Y., & Falkner, K. (2014, August). FLUSH+ RELOAD: A High Resolution, Low Noise, L3 Cache Side-Channel Attack. In USENIX Security Symposium (pp. 719-732).

[6] Cock, D., Ge, Q., Murray, T., & Heiser, G. (2014, November). The last mile: An empirical study of timing channels on sel4\. In Proceedings of the 2014 ACM SIGSAC Conference on Computer and Communications Security (pp. 570-581). ACM.

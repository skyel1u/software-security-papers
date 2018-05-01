# Timely Rerandomization for Mitigating Memory Disclosures

为缓解内存泄漏的运行时重随机化

Address Space Layout Randomization (ASLR) - 内存地址空间布局随机化。

ASLR的一个弱点是它假设储存地址的存储器是安全的，因此在面对存储器泄露漏洞不起作用，即使是细粒度的ASLR也被证明对内存泄露无效。

通过在每次产生输出时对进程的内存布局应用重随机化，这种方法会使得利用泄露信息的攻击者在劫持控制流的时候失效。Paper原型运行于C代码，重编译了程序且使用了一组增强信息来跟踪指针位置，且支持运行时的重随机化。

## Introduction



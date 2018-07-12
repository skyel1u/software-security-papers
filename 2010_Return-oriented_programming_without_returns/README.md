# Return-oriented programming without returns

## Abstract

这篇文章证明了在X86和ARM架构上都可能在不使用`ret`的情况下，来进行ROP。我们利用的是一些指令序列取代语义上的返回指令，而这些指令序列在X86(Linux)和ARM(Android)的大型库中出现的足够频繁，足以我们创建一个最小的图灵完备的Gadgets集合。

由于我们没有使用返回指令，我们提出的新型攻击对于最近提出的几种针对ROP的防御是有效的，因为那些防御手段检测指令流中频繁使用返回指令，或是检测栈中的地址是否为正常控制流的返回地址，或是修改编译器以免出现返回指令的保护手段。

## Introduction


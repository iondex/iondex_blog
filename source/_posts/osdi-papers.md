---
title: OSDI 文献概览
date: 2020-10-14 20:02:07
tags:
- 文献阅读
---

## OSDI'18

### REPT: Reverse Debugging of Failures in Deployed Software

微软Cui Weidong团队作品。

Binary级别的reverse debugging文章，思路是结合online的轻量级tracing（实现中是Intel PT）进行控制和信息记录，及离线的执行信息还原。

### Finding Crash-Consistency Bugs with Bounded Black-Box Crash Testing

分析文件系统的Crash，通过文件系统的黑盒fuzzing。基本思想是生成一串workload交由文件系统操作。

难点在于workload空间是无限的，解决方法是限制这个空间（比如说可以进行的操作、操作数量等）。

提到了一个insight，认为文件系统的crash绝大多数都可以在一个全新的文件系统上执行几步workload就可以触发。

文章发现了之前5年发现的24个Linux文件系统bug，且发现了10个新的bug。

### Graviton: Trusted Execution Environments on GPUs

在GPU上实现Trusted Execution（TE）的文章。卖点在于对现有系统的破坏比较小（只需要修改诸如GPU命令处理器等边缘设备）。

缺点在于overhead比较大（17%-33%），且overhead集中于发送及接收数据带来的加/解密运算。

### ZebRAM: Comprehensive and Compatible Software Protection Against Rowhammer Attacks

Rowhammer attack：由于现代内存（DRAM）中数据的存储单元（cell）密度极大，所以当非常频繁地读取一个单元的数据时有可能导致该单元的电压泄露，影响相邻的行（row）上的数据（比如导致位反转）。

Rowhammer attack会影响当前内存cell周围的**行**。因此ZebRAM使用纯软件的方式将内存行隔开以防止Rowhammer attack。

## OSDI'16

### Push-Button Verification of File Systems via Crash Refinement
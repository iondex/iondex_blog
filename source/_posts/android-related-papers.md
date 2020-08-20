---
title: Android Fuzzing/程序分析 相关文献
date: 2020-08-16 15:26:33
tags:
---

### IntelliDroid (NDSS'16)

IntelliDroid: A Targeted Input Generator for the Dynamic Analysis of Android Malware

研究对象：以触发某个特定API为目的，为动态分析工具产生**输入**。

开源：否。

方法：

1. 静态分析阶段：
   1. 对于一个指定的API，逆向分析从程序入口（某个Event Handler）到这个API的Call Path。可能有多个Call Path。
   2. 用类似符号执行的方法解出每个Path上的Constraint。
   3. 一个Call Path上可能有多个事件（Event Chains）。
2. 动态阶段：
   1. 获取运行时和每个约束相关的具体值。
   2. 注入事件。

### HARVESTOR (NDSS'16)

Harvesting Runtime Values in Android Applications That Feature Anti-Analysis Techniques

目的：获得运行时指定的某个值，由于Obfuscation/加壳/加密等其他原因静态分析不出来。

开源：否。

方法：静态 Backward Slicing。


### CopperDroid (NDSS'15)

CopperDroid: Automatic Reconstruction of Android Malware Behaviors

目的：理解/重建（Reconstruct） Malware 的行为。

开源：否。

方法：Syscall分析。

1. 采用插桩过的QEMU模拟器运行无修改的Android，对QEMU进行插桩记录系统调用（swi指令，CPU privilege-level transitions寄存器`cpsr`）。
2. 运行另一个Android虚拟机和特殊APP作为Unmarshalling Oracle，用来解码`io_ctl`中Binder调用的Payload。
3. 根据以上两步的结果，采用(Value-Based) Data Flow Analysis重建和分析Malware的行为。


### DroidScope (NDSS'12)

DroidScope: Seamlessly Reconstructing the OS and Dalvik Semantic Views for Dynamic Android Malware Analysis

目的：通过多层次（Multi-level）行为监控和分析，探究 Malware 行为并辅助分析。

开源：**是**。

方法：多层次分析（旧文章，不支持新的ART）。

1. OS层。
   1. Syscall：通过QEMU插桩，分析系统调用指令swi及其参数、返回值。
   2. Process/Thread：通过监控`task_struct`记录进程/线程信息。
   3. Memory Map：维护一个 Shadow `mm_struct` 结构，并插桩`sys_mmap2`更新进程的Memory Map信息。
2. Dalvik层。
   1. Dalvik支持JIT。通过分析Dalvik的Memory Map找出JVM指令和经过JIT编译的Hotspot机器码（涉及Dalvik底层）。
   2. 同样通过分析Memory Map找到Dalvik虚拟机状态。
   3. 通过`objdump`、Dalvik的Memory Map找到Symbol。


### DroidUnpack (NDSS'18)

Things You May Not Know About Android (Un)Packers: A Systematic Study based on Whole-System Emulation

目的：寻找一个通用的脱壳方法。DroidScope的后置工作。

背景：ART环境通过`dex2oat`将`dex`文件转换为`oat`机器码。（`oat`本质上是一个ELF，但有特殊的段）

方法：通过底层行为监控找出相关行为。

1. 在DroidScope的基础上做改进，重建OS-Leve和ART-Level的语义。
   1. OS-Level：找出所有进程的命名和Module加载信息，以便定位所有的Native Function。
   2. ART-Level：
      1. 对`libcutils.so`中的`set_process_name`函数插桩，确定进程的具体名称。
      2. 寻找进程中加载的`libart.so`，对于经过AOT编译的方法，ART会调用`ArtMethod::Invoke()`进行调用。
      3. 反之，会采用`DoCall()`调用Interpret模式的方法。从这两个途径找到`ArtMethod`结构。
      4. 从`ArtMethod::declaring_class_`找到在OAT文件中保存的原始DexFile和编译后的`OatClass`。
   3. 自此，所有Native和ART中动态加载的函数信息全部取出。
2. 代码行为分析。
   1. 监控所有写操作，记录Dirty Memory Region。遍历运行时的所有Basic Block，如果有重叠，说明这个Method是Unpack而来。
   2. 对Self Modifying Code和MultiLayer Pack的相关分析。


### FIRMSCOPE (NDSS'20)

FIRMSCOPE: Automatic Uncovering of Privilege-Escalation Vulnerabilities in Pre-Installed Apps in Android Firmware

目的：检测Firmware中预安装系统APK的权限提升（Privilege Escalation）问题。



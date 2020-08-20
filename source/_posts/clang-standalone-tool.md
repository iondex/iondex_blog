---
title: 编写Clang独立命令行工具
date: 2020-08-20 11:58:08
tags:
---

> 参考：
> 
> [Clang-Tutor](https://github.com/banach-space/clang-tutor)
> [Slide](https://s3.amazonaws.com/connect.linaro.org/yvr18/presentations/yvr18-223.pdf)
> [LibTooling](https://clang.llvm.org/docs/LibTooling.html)

LibTooling是LLVM提供的，用于编写基于Clang的独立命令行工具（Standalone Commandline Tool）的工具库。

和LibClang（提供一个基于Index和Cursor的C API）相比，LibTooling基于C++且功能更为强大，但由于功能更接近于Clang的核心，API更加复杂且不稳定，很可能随着Clang版本的变化导致API变化。
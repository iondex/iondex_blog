---
title: AOSP编译问题记录
date: 2020-08-16 13:09:00
tags: 
---

# 编译AOSP的准备工作

## 安装Repo

```bash
mkdir ~/bin
PATH=~/bin:$PATH
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo
```

修改`.bashrc`或`.zshrc`：

```bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
export LC_ALL=C
export PATH=~/bin:$PATH
```

## 安装Repo初始化包

``` bash
wget -c https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar # 下载初始化包
tar xf aosp-latest.tar
cd aosp
# Install Python
sudo apt-get install -y python
repo sync
```

这样设置之后`repo`的工作目录就在`aosp`文件夹，下载数据量为80G，`sync`后数据量为134G。

## 初始化编译目录

在**`aosp`目录内**新建一个文件夹，如`android-4.4.2_r1`，作为编译的输出目录。
初始化源码目录（`-b`后为目标版本）：

```bash
# init
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-4.4.2_r1
# sync
repo sync
```

注意：在`aosp`目录外执行`repo`则需要重新下载`repo`及其初始化组件。

## 准备编译环境

安装编译的依赖（摘自[Link](https://source.android.com/setup/build/initializing)）。

```bash
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip
```

## 安装旧版本Oracle JDK

对于AOSP来说，编译Android 5.0之前的版本需要**Oracle JDK 6**（OpenJDK会报错），安装方法如下：

从Java Archive Downloads [Link](https://www.oracle.com/java/technologies/javase-java-archive-javase6-downloads.html) 下载JDK版本。

当前下载需要登陆，网上的一个共享账号（目前此账号已经失效）：

```
账号：2696671285@qq.com 
密码：Oracle123
```

## 修改旧版本Make的限制

修改`build/core/main.mk`，加入新版本：

```makefile
# Add make-4.1
ifeq (0,$(shell expr $$(echo $(MAKE_VERSION) | sed "s/[^0-9\.].*//") = 4.1))
# ...
endif
```

## 编译Android 4.4.2

开始编译：

```bash
. build/envsetup.sh
lunch full_mako-userdebug 
m -j42
```

### 问题1

```
build/core/base_rules.mk:134: *** external/zlib: MODULE.TARGET.SHARED_LIBRARIES.libz already defined by external/arm-trusted-firmware/lib/zlib/zlib.  Stop.
```

解决方法：`external/arm-trusted-firmware/lib/zlib/zlib`是`external/zlib`的符号链接，这里并不清楚为何会导致此错误（猜测为`make`版本不正确导致无法正确检测符号链接）。删除`external/arm-trusted-firmware`即可继续编译。

### 问题2

```
make: *** No rule to make target 'hardware/qcom/sm8150/Android.mk'.  Stop.
```

（同样错误可能会出现在`hardware/`路径下的其他文件夹，解决方法类似）

解决方法：将文件夹`hardware/qcom/sm8150/`内的`Android.mk`和`Android.bp`文件改名。

### 问题3

```
Traceback (most recent call last):
  File "scripts/make_css_value_keywords.py", line 177, in <module>
    in_generator.Maker(CSSValueKeywordsWriter).main(sys.argv)
  File "/home/yufeng/aosp/external/chromium_org/third_party/WebKit/Source/core/scripts/in_generator.py", line 119, in main
    writer.write_files(options.output_dir)
  File "/home/yufeng/aosp/external/chromium_org/third_party/WebKit/Source/core/scripts/in_generator.py", line 77, in write_files
    self._write_file(output_dir, generator(), file_name)
  File "scripts/make_css_value_keywords.py", line 172, in generate_implementation
    gperf = subprocess.Popen(gperf_args, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
  File "/usr/lib/python2.7/subprocess.py", line 394, in __init__
    errread, errwrite)
  File "/usr/lib/python2.7/subprocess.py", line 1047, in _execute_child
    raise child_exception
OSError: [Errno 2] No such file or directory
Gyp action: third_party_WebKit_Source_core_core_derived_sources_gyp_make_derived_sources_target_MathMLNames (out/target/product/mako/obj/GYP/shared_intermediates/blink/MathMLNames.cpp)
external/chromium_org/third_party/WebKit/Source/core/make_derived_sources.target.linux-arm.mk:108: recipe for target 'out/target/product/mako/obj/GYP/shared_intermediates/blink/CSSValueKeywords.cpp' failed
make: *** [out/target/product/mako/obj/GYP/shared_intermediates/blink/CSSValueKeywords.cpp] Error 1
```

解决方式：安装`gperf`，`sudo apt-get install -y gperf`。

## 编译Android 6.0.1

安装OpenJDK 7，下载链接：

安装完成之后用`update-alternatives`调换版本。

注意：此版本的Makefile与当前下载的OpenJDK 7并不兼容，需要手动修改Makefile。

开始编译：

```bash
. build/envsetup.sh
lunch aosp_bullhead-userdebug 
m -j42
```

## 问题1

```
error: unsupported reloc 43
```

参考：https://stackoverflow.com/questions/36048358/building-android-from-sources-unsupported-reloc-43

此问题是由于提供的工具链`ld`版本不正确导致的，解决方式：

`cp /usr/bin/ld.gold prebuilts/gcc/linux-x86/host/*/x86_64-linux/bin/ld`

即将`repo`目录下`prebuilts/gcc/linux-x86/host`目录下所有的工具链内`ld`命令进行替换。

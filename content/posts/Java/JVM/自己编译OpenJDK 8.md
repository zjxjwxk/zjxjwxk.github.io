---
title: 自己编译OpenJDK 8
date: 2020-07-16 11:01:00
tags: 
- Java
- JVM
categories: 
- Java
preview: 300
---



想要一探 JDK 内部的实现机制，最便捷的捷径之一就是自己编译一套 JDK，通过阅读和跟踪调试 JDK 源码去了解 Java 技术体系的原理。本人选择了 OpenJDK 进行编译。

由于在编译 OpenJDK 7 时出现了如下不知如何解决的问题：

```bash
llvm-gcc -m64   -m64  -L`pwd`  -framework CoreFoundation  -o gamma launcher/java_md.o launcher/java.o launcher/jli_util.o launcher/wildcard.o -ljvm -lm -pthread
Undefined symbols for architecture x86_64:
  "_JNI_CreateJavaVM", referenced from:
      _LoadJavaVM in java_md.o
  "_JNI_GetDefaultJavaVMInitArgs", referenced from:
      _LoadJavaVM in java_md.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make[8]: *** [gamma] Error 1
make[7]: *** [the_vm] Error 2
make[6]: *** [product] Error 2
make[5]: *** [generic_build2] Error 2
make[4]: *** [product] Error 2
make[3]: *** [all_product_universal] Error 2
make[2]: *** [universal_product] Error 2
make[1]: *** [hotspot-build] Error 2
make: *** [build_product_image] Error 2
```

个人猜想是由于我 Mac OS 系统版本太高的问题（Catalina 10.15.5），XCode 版本也是最新的 11.6。

于是，我尝试编译更高版本的 OpenJDK 8，在解决一系列编译问题后，成功了。



## 获取 JDK 源码

获取 JDK 源码有两种方式：

（1）通过 Mercurial 代码版本管理工具从 Repository 中直接取得源码

Repository 地址：http://hg.openjdk.java.net/jdk8/jdk8

获取过程如以下代码所示

```bash
hg clone http://hg.openjdk.java.net/jdk8/jdk8
cd jdk7u-dev
chmod 755 get_source.sh
./get_source.sh
```

从版本管理中看变更轨迹效果较好，但不足之处是速度太慢，而且 Mercurial 不如 Git、SVN 等版本控制工具那样普及。

（2）通过 OpenJDK™ Source Releases 页面取得打包好的源码

页面地址：https://download.java.net/openjdk/jdk8/



## 系统需求

建议在 Linux、MacOS 或 Solaris 上构建 OpenJDK

本人采用的是 64 位操作系统，编译的也是 64 位的 OpenJDK



## 构建编译环境

本人使用的是 MacOS ，需要安装最新版本的 XCode 和 Command Line Tools for Xcode，另外还要准备一个 N-1 （N 为要编译的OpenJDK 版本号）以上版本的 JDK，官方称这个 JDK 为 “Bootstrap JDK” 。此处由于编译的是 OpenJDK 8，我选用 JDK 7 作为 Bootstrap JDK。最后，需要下载一个 1.7.1 以上版本的 Apache Ant，并添加环境变量，用于执行 Java 编译代码中的 Ant 脚本。用 brew 安装 freetype 和 CUPS。



## 进行编译

最后我们还需要对系统的环境变量做一些简单设置以便编译能够顺利通过，这里给出使用的编译 Shell 脚本。其中添加导出了一些 ALT_ 环境变量（在编译 OpenJDK 8 的时候警告 ALT_ 被弃用了，因此这里注释掉了），如 freetype 和 CUPS。

并将 COMPILER_WARNINGS_FATAL=false 以避免编译器的语法校验太严格：

```bash
#语言选项,这个必须设置,否则编译好后会出现一个HashTable的NPE错
export LANG=C
#Bootstrap JDK的安装路径。必须设置
#export ALT_BOOTDIR=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home

#允许自动下载依赖
export ALLOW_DOWNLOADS=true

#并行编译的线程数,设置为和CPU内核数量一致即可
export HOTSPOT_BUILD_J0BS=8
#export ALT_PARALLEL_COMPILE_JOBS=8

#比较本次build出来的映像与先前版本的差异。这对我们来说没有意义, 
#必须设置为false,香则sanity检查会报缺少先前版本JDK的映像的错误提示。 
#如桌已经设置dev或者DEV_ONLY=true,这个不显式设置也行
export SKIP_COMPARE_IMAGES=true

#使用预编译头文件,不加这个编译会更慢一些
export USE_PRECOMPILED_HEADER=true

#要编译的内容
export BUILD_LANGTOOLS=true
#export BUILD_JAXP=false
#export BUILD_JAXWS=fa1se
#export BUILD_CORBA=false
export BUILD_HOTSPOT=true
export BUILD_JDK=true

#要编译的版本
#export SKIP_DEBUG_BUILD=false
#export SKIP_FASTDEBUG_BUILD=true
#export DEBUG_NAME=debug

#把它设置为false可以避开javaws和浏览器Java插件之类的部分的build
BUILD_DEPLOY=false

#把它设置为false就不会build出安装包。因为安装包里有些奇怪的依赖, 
#但即便不build出它也已经能得到完整的JDK映像,所以还是别build它好了
BUILD_INSTALL=false

#编译结果所存放的路径
#export ALT_OUTPUTDIR=/Users/zjxjwxk/Documents/JVM/JDK-Build/build

#export ALT_FREETYPE_HEADERS_PATH=/usr/local/Cellar/freetype/2.6.2/include/freetype2
#export ALT_FREETYPE_LIB_PATH=/usr/local/Cellar/freetype/2.6.2/lib

#export ALT_CUPS_HEADERS_PATH=/usr/local/Cellar/cups/2.3.3/include

export COMPILER_WARNINGS_FATAL=false

#这两个环境变量必须去掉,不然会有很诡异的事情发生（我没有具体查过这些 "诡异的
#事情” ,Makefile脚本裣查到有这2个变量就会提示警告)
unset JAVA_HOME
unset CLASSPATH
make 2>&1 | tee build/build.log
```



### 检查

全部设置结束之后，可以输入 `make sanity` 来检查我们所做的设置是否全部正确。如果一切顺利，那么几秒钟之后会有类似代码清单1-2所示的输出。

```bash
~/Develop/JVM/jdkBuild/openjdk_7u4$ make sanity  
Build Machine Information:  
   build machine = IcyFenix-RMBP.local  
 
Build Directory Structure:  
   CWD = /Users/IcyFenix/Develop/JVM/jdkBuild/openjdk_7u4  
   TOPDIR = .  
   LANGTOOLS_TOPDIR = ./langtools  
   JAXP_TOPDIR = ./jaxp  
   JAXWS_TOPDIR = ./jaxws  
   CORBA_TOPDIR = ./corba  
   HOTSPOT_TOPDIR = ./hotspot  
   JDK_TOPDIR = ./jdk  
 
Build Directives:  
   BUILD_LANGTOOLS = true   
   BUILD_JAXP = true   
   BUILD_JAXWS = true   
   BUILD_CORBA = true   
   BUILD_HOTSPOT = true   
   BUILD_JDK    = true   
   DEBUG_CLASSFILES =    
   DEBUG_BINARIES =    
 
……因篇幅关系，中间省略了大量的输出内容……  
   
OpenJDK-specific settings:  
  FREETYPE_HEADERS_PATH = /usr/X11R6/include  
    ALT_FREETYPE_HEADERS_PATH =   
  FREETYPE_LIB_PATH = /usr/X11R6/lib  
    ALT_FREETYPE_LIB_PATH =   
 
Previous JDK Settings:  
  PREVIOUS_RELEASE_PATH = USING-PREVIOUS_RELEASE_IMAGE  
    ALT_PREVIOUS_RELEASE_PATH =   
  PREVIOUS_JDK_VERSION = 1.6.0  
    ALT_PREVIOUS_JDK_VERSION =   
  PREVIOUS_JDK_FILE =   
    ALT_PREVIOUS_JDK_FILE =   
  PREVIOUS_JRE_FILE =   
    ALT_PREVIOUS_JRE_FILE =   
  PREVIOUS_RELEASE_IMAGE = /Library/Java/JavaVirtualMachines/jdk1.7.0_04.jdk/Contents/Home  
    ALT_PREVIOUS_RELEASE_IMAGE =   
 
Sanity check passed.  
```

Makefile 的 Sanity 检查过程输出了编译所需的所有环境变量，如果看到 “Sanity check passed” 说明检查过程通过了，可以输入 `make` 执行整个 OpenJDK 编译 (make 不加参数，默认编译 make all)。



### 可能出现的编译错误

1. ```bash
   error: equality comparison with extraneous
   error: '&&' within '||'
   ```

   这是因为编译器语法校验太严格了，添加环境变量 `export COMPILER_WARNINGS_FATAL=false` 即可。

2. ```bash
   clang: error: unknown argument: '-fpch-deps'
   ```

   这是因为新的编译器已经不再支持这个选项了，打开 `hotspot/make/bsd/makefiles/gcc.make`，找到 `-fpch-deps` 所在的那一行，注释掉即可。

3. ```bash
   invalid argument '-std=gnu++98' not allowed with 'C/ObjC'
   ```

   在 `common/autoconf/generated-configure.sh` 中，注释掉 `CXXSTD_CXXFLAG="-std=gnu++98"` 这一行，然后重新执行 configure。

   ```bash
   ./configure  --with-freetype-include=/usr/local/Cellar/freetype/2.6.2/include/freetype2 --with-freetype-lib=/usr/local/Cellar/freetype/2.6.2/lib
   ```



### 编译完成

编译成功后，显示：

```bash
----- Build times -------
Start 2020-07-16 12:38:50
End   2020-07-16 12:48:23
00:00:17 corba
00:05:36 hotspot
00:00:11 jaxp
00:00:19 jaxws
00:02:44 jdk
00:00:25 langtools
00:09:33 TOTAL
-------------------------
Finished building OpenJDK for target 'default'
```



### 运行 java 命令

编译完成后，进入 OpenJDK 源码下的 `build/macosx-x86_64-normal-server-release/jdk` 目录，这是整个 JDK 的完整编译结果，复制到 JAVA_HOME 目录，就可以作为一个完整的 JDK 使用，编译出来的虚拟机，在 `-version` 命令中带有用户的机器名。

```bash
> ./java -version  
openjdk version "1.8.0-internal"
OpenJDK Runtime Environment (build 1.8.0-internal-zjxjxk_2020_07_16_13_03-b00)
OpenJDK 64-Bit Server VM (build 25.71-b00, mixed mode)
```



### 可能出现的运行问题

我这里运行时出现了一个类似以下的问题：

```shell
Error: A fatal exception has occurred. Program will exit.
localhost:bin jjchen$ ./java -version
openjdk version "1.8.0-internal"
OpenJDK Runtime Environment (build 1.8.0-internal-jjchen_2018_09_13_10_00-b00)
OpenJDK 64-Bit Server VM (build 25.71-b00, mixed mode)
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGILL (0x4) at pc=0x0000000107487f47, pid=88445, tid=0x0000000000002603
#
# JRE version: OpenJDK Runtime Environment (8.0) (build 1.8.0-internal-jjchen_2018_09_13_10_00-b00)
# Java VM: OpenJDK 64-Bit Server VM (25.71-b00 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# V  [libjvm.dylib+0x487f47]
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /Users/jjchen/jvm/jdk8u-dev/build/macosx-x86_64-normal-server-release/jdk/bin/hs_err_pid88445.log
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
#
```

在将 `hotspot/src/share/vm/runtime/perfData.cpp` 中的 `delete p` 注释掉，并重新编译后，得以正常运行。

```c++
void PerfDataManager::destroy() {

  if (_all == NULL)
    // destroy already called, or initialization never happened
    return;

  for (int index = 0; index < _all->length(); index++) {
    PerfData* p = _all->at(index);
    // delete p;
  }

  delete(_all);
  delete(_sampled);
  delete(_constants);

  _all = NULL;
  _sampled = NULL;
  _constants = NULL;
}
```



## 单独编译 HotSpot 虚拟机

在大多数时候，如果我们不关心 JDK 中 HotSpot 虚拟机以外的内容，只想单独编译 HotSpot 虚拟机的话（例如调试虚拟机时，每次改动程序都执行整个 OpenJDK 的Makefile，速度肯定受不了），那么使用 hotspot/make 目录 下的 Makefile 进行替换即可，其他参数设置与前面是一致的，这时候虚拟机的输出结果存放在 `build/macosx-x86_64-normal-server-release/hotspot/bsd_amd64_compiler2` 目录中，进入后可以见到以下几个目录。

```bash
drwxr-xr-x   11 zjxjwxk  staff    352  7 16 13:04 debug
drwxr-xr-x   11 zjxjwxk  staff    352  7 16 13:04 fastdebug
drwxr-xr-x   17 zjxjwxk  staff    544  7 16 13:04 generated
drwxr-xr-x   11 zjxjwxk  staff    352  7 16 13:04 optimized
drwxr-xr-x  643 zjxjwxk  staff  20576  7 16 13:09 product
```

这些目录对应了不同的优化级别，优化级别越高，性能自然越好，但是输出代码与源码的差别就越大，难于调试，具体哪个目录有内容，取决于 `make` 命令后面的参数。

在编译结束之后、运行虚拟机之前，还要手工编辑目录下的 env.sh 文件，这个文件由编译脚本自动产生，用于设置虚拟机的环境变量，里面已经发布了 “JAVA_HOME、CLASSPATH、HOTSPOT_BUILD_USER” 3个环境变量，还需要增加一个“LD_LIBRARY_PATH”，内容如下：

```bash
LD_LIBRARY_PATH=.:${JAVA_HOME}/jre/lib/amd64/native_threads:${JAVA_HOME}/jre/lib/amd64:  
export LD_LIBRARY_PATH 
```

然后执行以下命令启动虚拟机（这时的启动器名为gamma），输出版本号。

```bash
. ./env.sh  
./gamma -version  
Using java runtime at: /Library/Java/JavaVirtualMachines/jdk1.7.0_04.jdk/Contents/Home/jre  
java version "1.7.0_04"  
Java(TM) SE Runtime Environment (build 1.7.0_04-b21)  
OpenJDK 64-Bit Server VM (build 23.0-b21, mixed mode) 
```

看到自己编译的虚拟机成功运行起来，很有成就感吧!



> 参考：《深入理解Java虚拟机：JVM高级特性与最佳实践（第2版）》
>
> 作者：周志明
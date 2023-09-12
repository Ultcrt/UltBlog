---
title: 同时链接 Ipopt 与 Open3D 导致 Segment Fault 的原因分析与解决方案
date: 2023-09-12T02:48:37.343Z
last_modified_at: 2023-09-12T02:48:37.351Z
excerpt: 开发项目时发现，同时链接  coin-or/Ipopt 和 Open3D 会在运行时抛出 Segment Fault，在 Ipopt
  开发者的指点下，发现导致问题的原因是两者依赖的 BLAS 库冲突，需要使用静态 BLAS 库编译 Ipopt。本文对导致该问题的原因进行了记录，并对使用静态
  BLAS 库编译 Ipopt 的方法进行了探索。
categories:
  - 工作随笔
tags:
  - 编译
  - Ipopt
  - Open3D
  - Libtool
  - BLAS
  - LAPACK
header:
  overlay_image: https://picsum.photos/1920/640
  caption: "来源: [**Lorem Picsum**](https://picsum.photos/)"
  teaser: /assets/images/site/default-teaser.png
---
以下内容总结自本人与 Ipopt 开发者的[讨论](https://github.com/coin-or/Ipopt/discussions/694)。

## 问题描述

在开发项目时，把 Ipopt 和 Open3D 链接到了同一个 C++ 项目中，使用的CMake代码如下所示：
```cmake
cmake_minimum_required(VERSION 3.26)
project(link_test)

set(CMAKE_CXX_STANDARD 17)

find_package(Open3D REQUIRED)

add_executable(link_test main.cpp myTNLP.cpp myTNLP.h)

target_link_libraries(link_test PRIVATE ipopt Open3D::Open3D)   # Segment fault occurs
```
编译一切正常，但是在运行时程序给出了 Segment Fault，如下图所示（接取自Clion Debugger）：

![](/UltBlog/assets/images/uploads/266275784-34080407-9912-4982-9871-0200f7901196.png)

但是 Ipopt 与 Open3D 都能够正常独立运行，只有在将两个库链接在一个项目时，会出现该问题。

## 原因
在咨询 Ipopt 开发者后，他提到由于 Segment Fault 是 MKL 库执行时产生的，所以问题很有可能是由于 Open3D 使用的 MKL 库与 Ipopt 使用的 BLAS 库冲突导致的，例如 Ipopt 使用32位整型接口，如果 Open3D 使用的 MKL 库是64位整型的，就会导致冲突（MKL 中包含 BLAS 的实现）。因此使用静态 BLAS 库构建 Ipopt 应该能够解决该问题。

我按照 Ipopt 开发者给出的思路，通过阅读 [Open3D 的 CMake 代码](https://github.com/isl-org/Open3D/blob/master/3rdparty/mkl/mkl.cmake#L155) ，发现 Open3D 确实使用了64位整型的MKL库（libmkl_intel_ilp64.a）。

## 解决方案
### OpenBLAS
在配置Ipopt时使用下述标志：
```shell
--with-lapack-lflags="-Wl,--no-as-needed /opt/OpenBLAS/lib/libopenblas.so -lm"
```
即可得到使用静态 OpenBLAS 库作为 BLAS 的 Ipopt 版本。将该版本的 Ipopt 与 Open3D 链接到同一个项目不会出现 Segment Fault。
### MKL
在配置Ipopt时使用从 [MKL link line advisor](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-link-line-advisor.html#gs.5jnf7s) 工具中得到的标志：
```shell
--with-lapack-lflags="-Wl,--start-group \
${MKLROOT}/lib/intel64/libmkl_intel_lp64.a \
${MKLROOT}/lib/intel64/libmkl_gnu_thread.a \
${MKLROOT}/lib/intel64/libmkl_core.a -Wl,--end-group -lgomp -lpthread -lm -ldl"
```
然而使用上述标志虽然能够正常编译出 Ipopt 库，但是在编译 Ipopt 的测试代码时，链接器抛出了大量的 Undefined Reference 报错：
```shell
/usr/bin/ld: ../src/.libs/libipopt.so: undefined reference to `mkl_pds_lp64_ch_blkldlslvs_ooc_pardiso'
/usr/bin/ld: ../src/.libs/libipopt.so: undefined reference to `mkl_blas_lp64_zscal'
...
/usr/bin/ld: ../src/.libs/libipopt.so: undefined reference to `mkl_pds_pds_her_indef_bk_fct_slv_cmplx'
/usr/bin/ld: ../src/.libs/libipopt.so: undefined reference to `mkl_pds_lp64_sp_pds_copy_a2l_value_omp_cmplx'
```
但是，此时如果将 Open3D 使用的合并版本的 MKL 库链接到 Ipopt 中（Open3D进行合并操作是为了解决循环依赖的问题，详见 [Open3D 的 Cmake](https://github.com/isl-org/Open3D/blob/master/3rdparty/mkl/mkl.cmake#L146)），Undefined Reference 报错消失。
因此我决定使用 [Open3D 生成合并库的方式](https://github.com/isl-org/Open3D/blob/master/3rdparty/mkl/mkl.cmake#L170)，生成 Ipopt 所使用的、合并后的MKL静态库，代码如下所示：
```shell
ar x /usr/lib/x86_64-linux-gnu/libmkl_intel_lp64.a
ar x /usr/lib/x86_64-linux-gnu/libmkl_gnu_thread.a
ar x /usr/lib/x86_64-linux-gnu/libmkl_core.a

find -name "*.o" | xargs ar -qc ./lib/libmkl_merged.a
```
使用下述标志即可使用合并后的静态库构建Ipopt：
```shell
--with-lapack-lflags="-Wl,--no-as-needed /path/to/merged/lib/libmkl_merged.a -lgomp -lpthread -lm -ldl"
```
构建得到的新 Ipopt 库，可以正常的与 Open3D 进行链接，能够正常使用。

但是，上述过程无法解释为什么通过 MKL link line advisor 生成的指令构建的 Ipopt 库会出现 Undefined Reference 的问题。

在调查一段时间但是没有结果后，我再次询问了 Ipopt 开发者，他指出，该问题是由于 libtool 会重构链接标志的次序，使得静态库不再位于 `--start-group` 与 `--end-group` 之间，从而导致循环依赖无法被解决。而将静态库合并成一个，在解决了循环依赖的同时绕开了该问题，因此生成的 Ipopt 库能够正常使用。

按照 Ipopt 开发者给出的思路，我通过下述指令查看了 libtool 实际生成的链接指令：
```shell
make V=1 | grep link
```
发现使用 MKL link line advisor 给出的指令构建 Ipopt 时的实际链接指令为：
```shell
g++ ... libmkl_intel_lp64.a libmkl_gnu_thread.a libmkl_core.a -lgomp -lpthread -lm -ldl \
-Wl,--no-as-needed -Wl,--start-group -Wl,--end-group ...
```
可以看到，静态库被调整到 `--start-group` 与 `--end-group` 之外，因此 `--start-group` 与 `--end-group` 没有起到实际作用，无法解决循环依赖。

在阅读[此 Debian 的 Bug 反馈网页](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=159760&repeatmerged=yes)(该网页同样给出了 libtool 调整标志次序做法的合理性)后，我了解到了解决该方法的一种heck（使用逗号连接各个链接器标志）：
```shell
--with-lapack-lflags="-Wl,--no-as-needed -Wl,--start-group,\
${MKLROOT}/lib/intel64/libmkl_intel_lp64.a,\
${MKLROOT}/lib/intel64/libmkl_gnu_thread.a,\
${MKLROOT}/lib/intel64/libmkl_core.a,--end-group \
-lgomp -lpthread -lm -ldl"
```
使用上述标志，libtool就能够将正确的标志次序传输给链接器，从而解决循环依赖的问题，生成能够正常使用的 Ipopt 库。

## 总结
1. 解决链接相关问题时，需要从实际的链接命令入手，上层封装很难发现问题本质。
1. 在难以推进时进行求助，可能能够获得很好的思路，从而快速解决问题（当然不能过于依赖求助）。























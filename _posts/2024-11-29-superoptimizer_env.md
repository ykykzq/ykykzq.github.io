---
title: '超级优化环境配置'
date: 2024-11-29
permalink: /posts/2024/11/blog-post-2/
tags:
  - 超级优化
  - 环境配置
---


关于超级优化项目环境配置的详细文档
# 超级优化环境配置

## 1. 安装Java环境

Java版本需大于等于Java11，可以利用以下指令安装

```
sudo apt install openjdk-11-jdk
```

## 2. 安装 `libcurl` 开发包

项目依赖`CURL`库，可以通过以下指令安装

```
sudo apt install libcurl4-openssl-dev
```

## 3. 安装Boost库

项目依赖于`Boost`库，可以通过以下指令安装

```
sudo apt install libboost-all-dev
```

## 4. 安装Z3

从z3的仓库下载源代码并安装：

```
git clone https://github.com/Z3Prover/z3.git
```

> 不可以通过`sudo apt install libz3-dev`直接安装z3，否则可能提示找不到"bit2bool"函数，构建失败。

然后切换进仓库目录，执行

```
python scripts/mk_make.py
```

之后执行

```
cd build
make
```

执行完整之后，执行如下指令安装即可

```
sudo make install
```

## 5. 编译项目

初次编译之前，应当执行如下指令更新Submodules：

```
git submodule update --init --recursive
```

以后在编译之前也应该执行如下指令：

```
git submodule update --remote
```

然后一路按照教程编译即可：

```
mkdir build
cd build/
cmake ..
make -j
```

> 如果出现错误，例如内存不足（c++: fatal error: Killed signal terminated program cc1plus），可以执行`make -j4`，减少并发线程数
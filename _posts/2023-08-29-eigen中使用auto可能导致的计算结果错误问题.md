---
title: Eigen中使用auto可能导致的计算结果错误问题
date: 2023-08-29T01:59:32.916Z
last_modified_at: 2023-08-29T07:52:47.044Z
excerpt: 在Eigen中使用auto导致向量类型被识别为 CwiseBinaryOp
  ，该对象相当于存储了运算公式而非计算结果，进而由于变量更新或悬空引用问题，导致运算结果异常，甚至报错。
categories:
  - 工作随笔
tags:
  - Eigen
  - C++陷阱
header:
  overlay_image: https://picsum.photos/1920/640
  caption: "来源: [**Lorem Picsum**](https://picsum.photos/)"
  teaser: https://eigen.tuxfamily.org/dox/Eigen_Silly_Professor_64x64.png
---
## 背景
项目开发过程中，使用Eigen计算一条路径的中点时发现计算结果存在问题，计算的结果并不在路径上。

起初，由于计算结果使用过Open3D进行可视化显示，因此怀疑是Open3D的可视化接口存在问题导致的。但是在使用简单数据进行进一步测试发现，Eigen库的计算结果存在问题，而非可视化接口的显示问题。

## 复现
复现问题的简化代码如下所示：
```c++
vector<Eigen::Vector3d> line;

Eigen::Vector3d v0(0, 1, 2), v1(0, -1, 2);

const auto v2 = (v0 - v1).normalized() * 0.5;
std::cout << "A: " << v2  << std::endl;

line.insert(line.begin(), v2);
std::cout << "B: " << v2 << std::endl;
```
上述代码的执行结果为（为了直观显示结果，博文对数据进行了简单的格式化处理）：
```shell
> A: [0,  0.5, 0]

> B: [0, 0.25, 0]    // 预期输出应与A相同，与预期不符
```
可以看到，两次输出之间并没有对 v2 进行修改，但是输出结果却不一致，与预期不符。

## 分析
在进行调查后发现，该问题是由于滥用了 auto 关键字导致编译器将向量/矩阵的运算结果识别为 CwiseBinaryOp 类型。该类型相当于存储了**二元运算与两个运算变量的引用**，而非运算结果，仅在需要获取结果时才会进行计算。这种非立刻计算结果的运算方式被称为 lazy evaluation（感觉有些类似符号运算），Eigen 使用这种方式提高运算性能。

但是，lazy evaluation 也带来了两个陷阱，分别为变量更新和悬空引用。

### 变量更新陷阱
由于 CwiseBinaryOp 结果并不是立刻计算的，因此如果修改 CwiseBinaryOp 对应的公式中包含的变量，CwiseBinaryOp 的计算结果也会随之改变，例如下述代码：
```c++
Eigen::Vector3d v0(0, 1, 2), v1(0, -1, 2);

const auto v2 = v0 + v1;    // 实际类型为 

std::cout << "A: " << v2 << std::endl;

v0 *= 2;

std::cout << "B: " << v2 << std::endl;
```
上述代码的执行结果为：
```shell
> A: [0, 0, 4]

> B: [0, 1, 6]
```
可以看到，上述代码在定义 v2 这一 CwiseBinaryOp 之后，对 v0 进行了修改，v2 的计算结果也随之更改。

### 悬空引用陷阱
由于 CwiseBinaryOp 中记录了对应公式中变量的引用，因此如果变量为暂存变量，在执行完 CwiseBinaryOp 对象的定义代码时，暂存变量被释放，就会出现悬空引用的问题，导致计算结果错误或者访存错误，例如下述代码：
```c++
Eigen::Vector3d v0(0, 1, 2), v1(0, -1, 2);

const auto v2 = (v0 - v1).normalized() * 0.5;
```
在定义 v2 的过程中，产生了 (v0 - v1).normalized() 这一暂存变量，因此类型为 CwiseBinaryOp 的 v2 将会记录这一暂存变量的引用。但是在 v2 定义执行完成后，该暂存变量被释放，导致 v2 中出现悬空引用。根据具体编译和运行环境，可能导致计算结果异常或程序抛出访存错误。

本博文给出在复现章节给出的异常代码即为悬空引用陷阱的复现。

## 结论
使用 Eigen 进行矩阵运算时，不要使用 auto 关键字，从而避免计算错误或是程序报错。
> *"In short: do not use the auto keywords with Eigen's expressions, unless you are 100% sure about what you are doing. In particular, do not use the auto keyword as a replacement for a Matrix<> type."*  
> 出处：[C++11 and the auto keyword \| Common Pitfalls \| Eigen Document](https://eigen.tuxfamily.org/dox/TopicPitfalls.html#title3)

## 参考资料
[1] [Wrong results using auto with Eigen \| StackOverflow](https://stackoverflow.com/questions/31099246/wrong-results-using-auto-with-eigen)

[2] [Erroneous use of auto type specifier with Eigen objects \| StackOverflow](https://stackoverflow.com/questions/58957421/erroneous-use-of-auto-type-specifier-with-eigen-objects)

[3] [Eigen evaluation with an auto hits temporal \| StackOverflow](https://stackoverflow.com/questions/72833132/eigen-evaluation-with-an-auto-hits-temporal)

[4] [Common Pitfalls \| Eigen Document](https://eigen.tuxfamily.org/dox/TopicPitfalls.html)
















































































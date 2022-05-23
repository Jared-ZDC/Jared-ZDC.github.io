---
layout: post
title: linux "__context__" 属性含义
categories: wiki
description: linux context 属性含义
keywords: linux context
---

# 背景
在分析Linux RCU lock的时候，发现每次在rcu_read_lock/rcu_read_unlock的时候，都会成对出现__acquire/__release接口，这两个接口的定义都是__context__，但是在内核中并没有找到出处。

# 备忘
查阅了相关资料，原来这个东西是给Linux代码静态检查工具使用的，为了保证lock/unlock必须成对出现使用的。
|  **宏名称**   | **宏定义**  | **检查点** |
|  ----  | ----  | ---- |
| __bitwise | __attribute__((bitwise)) | 确保变量是相同的位方式(比如 bit-endian, little-endiandeng) |
| __user | __attribute__((noderef, address_space(1))) | 指针地址必须在用户地址空间 |
| __kernel	| __attribute__((noderef, address_space(0))) | 指针地址必须在内核地址空间 |
| __iomem | __attribute__((noderef, address_space(2))) | 指针地址必须在设备地址空间 |
| __safe | __attribute__((safe)) | 变量可以为空 |
| __force | __attribute__((force)) | 变量可以进行强制转换 |
| __nocast | __attribute__((nocast)) | 参数类型与实际参数类型必须一致 |
| __acquires(x)	| __attribute__((context(x, 0, 1))) | 参数x 在执行前引用计数必须是0,执行后,引用计数必须为1 |
| __releases(x)	| __attribute__((context(x, 1, 0))) | 与 __acquires(x) 相反 |
| __acquire(x) | __context__(x, 1) | 参数x 的引用计数 + 1 |
| __release(x) | __context__(x, -1) | 与 __acquire(x) 相反 |
| __cond_lock(x,c)	| ((c) ? ({ __acquire(x); 1; }) : 0)	| 参数c 不为0时,引用计数 + 1, 并返回1 |

其中 __acquires(x) 和 __releases(x), __acquire(x) 和 __release(x) 必须配对使用, 否则 Sparse 会给出警告


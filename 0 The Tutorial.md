## 0 教程

### 0.1 介绍

Talloc 是一个具有析构函数的分层、引用计数的内存池系统。它建立在 C 标准库之上，定义了一组实用函数，这些函数完全简化了数据的分配和释放，尤其是对于包含许多动态分配元素（如字符串和数组）的复杂结构。
该库的主要目标是：消除为每个复杂结构创建清理函数的需要，提供已分配内存块的逻辑组织，并降低在长时间运行的应用程序中造成内存泄漏的可能性。所有这些都是通过在 talloc 上下文的层次结构中分配内存来实现的，这样递归地释放一个上下文也会释放其所有子上下文。

### 0.2 主要特点

- 一个开源项目
- 分层内存模型
- 数据结构到内存空间的自然投影
- 简化大型数据结构的内存管理
- 在内存释放之前自动执行析构函数
- 模拟动态类型系统
- 实现透明内存池

### 0.3 Table of contents:

Chapter 1: Talloc context

Chapter 2: Stealing a context

Chapter 3: Dynamic type system

Chapter 4: Using destructors

Chapter 5: Memory pools

Chapter 6: Debugging

Chapter 7: Best practises

Chapter 8: Using threads with talloc
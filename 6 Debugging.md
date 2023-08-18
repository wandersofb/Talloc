## 6 Debugging

尽管 talloc 使内存管理明显比 c 标准库更容易，但开发人员仍然只是人，可能会犯错误。

因此，了解一些用于检查 talloc 内存使用情况的工具会很方便。

### 6.1 Talloc 日志和中止

我们已经在“动态类型系统”一节中遇到了中止函数。在这种情况下，当检测到类型不匹配时使用它。然而，talloc 在其他几种情况下调用此中止函数：
- 当提供的指针不是有效的 talloc 上下文时，
- 当元数据无效时（可能是由于内存损坏），
- 以及当检测到空闲后的访问时。


第三个可能是最有趣的。它可以帮助我们通过 talloc 函数检测到试图双重释放上下文或任何其他操作（将其用作父对象、窃取它等）。

在释放上下文之前，talloc 会在元数据中设置一个标志。然后，这将用于检测空闲后的访问。它基本上是基于这样一种假设，即使正确地释放了内存，内存也会保持不变（至少在一段时间内）。即使用 TALLOC_FREE_FILL 环境变量中指定的值填充内存，这也会起作用，因为它只填充数据部分，而保留元数据不变。

除了中止功能外，talloc 还使用日志功能为上述违规行为提供额外信息。要启用日志记录，我们将使用以下之一设置日志函数：
- talloc_set_log_fn()
- talloc_set_log_stder()

以下代码是释放上下文后访问上下文的示例输出：

```c
talloc_set_log_stderr();
TALLOC_CTX *ctx = talloc_new(NULL);

talloc_free(ctx);
talloc_free(ctx);

results in:
talloc: access after free error - first free may be at ../src/main.c:55
Bad talloc magic value - access after free
```

另一个例子是无效上下文：

```c
talloc_set_log_stderr();
TALLOC_CTX *ctx = talloc_new(NULL);
char *str = strdup("not a talloc context");
talloc_steal(ctx, str);

results in:
Bad talloc magic value - unknown value
```

### 6.2 内存使用情况报告

Talloc 可以将指定 Talloc 上下文的内存使用情况报告打印到文件（到 stdout 或 stderr）。报告可以是简单的，也可以是完整的。简单报告仅提供有关上下文本身及其直系后代的信息。完整的报告递归地遍历整个上下文树。请参阅：
- talloc_report()
- talloc_report_full()

我们将使用以下代码来检索示例报告：

```c
struct foo {
  char *str;
};

TALLOC_CTX *ctx = talloc_new(NULL);
char *str =  talloc_strdup(ctx, "my string");
struct foo *foo = talloc_zero(ctx, struct foo);
foo->str = talloc_strdup(foo, "I am Foo");
char *str2 = talloc_strdup(foo, "Foo is my parent");

/* print full report */
talloc_report_full(ctx, stdout);
```

它将把 ctx 的完整报告打印到标准输出中。消息应类似于：

```c
full talloc report on 'talloc_new: ../src/main.c:82' (total 46 bytes in 5 blocks)
  struct foo contains 34 bytes in 3 blocks (ref 0) 0x1495130
    Foo is my parent contains 17 bytes in 1 blocks (ref 0) 0x1495200
    I am Foo contains 9 bytes in 1 blocks (ref 0) 0x1495190
  my string contains 10 bytes in 1 blocks (ref 0) 0x14950c0
```

我们可以在这个报告中注意到，包含结构体 foo 的上下文出现了问题。我们知道这个结构只有一个字符串元素。然而，我们可以从报告中看到，它有两个孩子。这表明我们要么违反了内存层次结构，要么忘记将其作为临时数据释放。查看代码，我们可以看到“Foo 是我的父母”应该附加到 ctx。
另请参阅：

- talloc_enable_null_tracking()
- talloc_disable_null_tracking()
- talloc_enable_leak_report()
- talloc_enable_leak_report_full()


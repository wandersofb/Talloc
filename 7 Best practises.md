## 7 Best practises

以下部分包含 Samba 和 SSSD 开发人员多年来发现的几个最佳实践和良好方式。

这些将帮助您编写更好、更容易调试并且尽可能少（希望没有）内存泄漏的代码。

### 7.1 保持上下文层次结构稳定

talloc 是一个层次结构内存分配器。层次结构的性质使编程更加防错。它使内存更易于管理和释放。因此，我们首先应该考虑的是：始终将数据结构投影到 talloc 上下文层次结构中。

这意味着，如果我们有一个结构，我们应该始终将其用作其元素的父上下文。这样，我们在释放结构或更改其父结构时就不会遇到任何麻烦。同样的规则也适用于数组。

例如，talloc 上下文层次结构部分的结构用户应该使用下一个图像上所示的上下文层次结构来创建。

![](.assert/context_tree.png)

### 7.2 每个函数都应该使用自己的上下文

在函数开头创建一个临时 talloc 上下文，并在 return 语句之前释放该上下文，这是一种很好的做法。所有数据都必须在此上下文或其子上下文上进行分配。这确保了只要我们不忘记释放临时上下文，就不会产生内存泄漏。

这种模式适用于这两种情况——当函数不返回任何动态分配的值时，以及当它返回时。然而，对于后一种情况，它需要一点扩展。

#### 7.2.1 不返回任何动态分配的函数

value

如果函数不返回在堆上创建的任何值，我们将只遵循前面提到的模式。

```C
int bar()
{
  int ret;
  TALLOC_CTX *tmp_ctx = talloc_new(NULL);
  if (tmp_ctx == NULL) {
    ret = ENOMEM;
    goto done;
  }
  /* allocate data on tmp_ctx or on its descendants */
  ret = EOK;
done:
  talloc_free(tmp_ctx);
  return ret;
}
```

#### 7.2.2 函数返回动态分配的值

如果我们的函数返回任何动态分配的数据，那么它的第一个参数应该始终是目标 talloc 上下文。此上下文充当输出值的父级。但是，我们将再次创建输出值作为临时上下文的后代。如果一切顺利，我们将把输出值的父级从临时上下文更改为目标 talloc 上下文。

此模式确保在发生错误（例如I/O错误或内存不足）时，释放所有分配的数据，并且在目标上下文中不会出现垃圾。

```C
int struct_foo_init(TALLOC_CTX *mem_ctx, struct foo **_foo)
{
  int ret;
  struct foo *foo = NULL;
  TALLOC_CTX *tmp_ctx = talloc_new(NULL);
  if (tmp_ctx == NULL) {
    ret = ENOMEM;
    goto done;
  }
  foo = talloc_zero(tmp_ctx, struct foo);
  /* ... */
  *_foo = talloc_steal(mem_ctx, foo);
  ret = EOK;
done:
  talloc_free(tmp_ctx);
  return ret;
}
```

### 7.3 在 NULL 上分配临时上下文

从上一个列表中可以看出，我们没有直接在 mem_ctx 上分配临时上下文，而是使用 NULL 作为 talloc_new() 函数的参数创建了一个新的顶级上下文。请看以下示例：

```C
char *create_user_filter(TALLOC_CTX *mem_ctx,
                         uid_t uid, const char *username)
{
  char *filter = NULL;
  char *sanitized_username = NULL;
  /* tmp_ctx is a child of mem_ctx */
  TALLOC_CTX *tmp_ctx = talloc_new(mem_ctx);
  if (tmp_ctx == NULL) {
    return NULL;
  }

  sanitized_username = sanitize_string(tmp_ctx, username);
  if (sanitized_username == NULL) {
    talloc_free(tmp_ctx);
    return NULL;
  }

  filter = talloc_aprintf(tmp_ctx,"(|(uid=%llu)(uname=%s))",
                          uid, sanitized_username);
  if (filter == NULL) {
    return NULL; /* tmp_ctx is not freed */ (*@\label{lst:tmp-ctx-3:leak}@*)
  }

  /* filter becomes a child of mem_ctx */
  filter = talloc_steal(mem_ctx, filter);
  talloc_free(tmp_ctx);
  return filter;
}
```

我们忘记在 filter==NULL 条件中的 return 语句之前释放 tmp_ctx。然而，它是作为 mem_ctx 上下文的子级创建的，因此，一旦释放 mem_ctx，它就会被释放。因此，不会产生可检测的内存泄漏。

另一方面，我们没有任何方法来访问分配的数据，而且据我们所知，mem_ctx 可能在应用程序的整个生命周期内都存在。由于这些原因，这应该被视为内存泄漏。我们如何检测它是否未被引用，但仍附加到其父上下文？唯一的方法是注意源代码中的错误。

但是，如果我们将临时上下文创建为顶级上下文，它将不会被释放，内存诊断工具（例如 valgrind）也能够完成它们的工作。

### 7.4 临时上下文和 talloc 池

如果我们想利用 talloc 池的优势，同时保持上一节中介绍的模式，我们无法直接做到这一点。最好的做法是创建一个条件构建，在这里我们可以决定如何创建临时上下文。例如，我们可以创建以下宏：

```C
#ifdef USE_POOL_CONTEXT
  #define CREATE_POOL_CTX(ctx, size) talloc_pool(ctx, size)
  #define CREATE_TMP_CTX(ctx)        talloc_new(ctx)
#else
  #define CREATE_POOL_CTX(ctx, size) talloc_new(ctx)
  #define CREATE_TMP_CTX(ctx)        talloc_new(NULL)
#endif
```

现在，如果我们的应用程序正在开发中，我们将使用未定义的宏 USE_POOL_CONTEXT 来构建它。通过这种方式，我们可以使用内存诊断实用程序来检测内存泄漏。

发布版本将使用定义的宏进行编译。这将启用池上下文，从而减少 malloc() 调用，从而加快处理速度。

```C
int struct_foo_init(TALLOC_CTX *mem_ctx, struct foo **_foo)
{
  int ret;
  struct foo *foo = NULL;
  TALLOC_CTX *tmp_ctx = CREATE_TMP_CTX(mem_ctx);
  /* ... */
}

errno_t handle_request(TALLOC_CTX mem_ctx)
{
  int ret;
  struct foo *foo = NULL;
  TALLOC_CTX *pool_ctx = CREATE_POOL_CTX(NULL, 1024);
  ret = struct_foo_init(mem_ctx, &foo);
  /* ... */
}
```
## 3 Dynamic type system

### 3.1 Dynamic Type System

用 C 语言进行泛型编程是非常困难的。面向对象语言既没有继承，也没有已知的模板。没有动态类型系统。因此，这种语言中的泛型编程通常是通过将变量类型转换为 void* 并通过泛型函数将其传递到专用回调来完成的，如下一个列表所示。

```C
void generic_function(callback_fn cb, void *pvt)
{
  /* do some stuff and call the callback */
  cb(pvt);
}

void specific_callback(void *pvt)
{
  struct specific_struct *data;
  data = (struct specific_struct*)pvt;
  /* ... */
}

void specific_function()
{
  struct specific_struct data;
  generic_function(callback, &data);
}
```

不幸的是，由于这种类型转换，类型信息丢失了。编译器在编译过程中无法检查类型，我们也无法在运行时进行检查。向回调提供无效的数据类型将导致应用程序出现意外行为（不一定是崩溃）。这个错误通常很难察觉，因为它不是第一个想到的。

正如我们已经知道的，每个 talloc 上下文都包含一个名称。这个名称在任何时候都可用，即使我们丢失了变量的类型，它也可以用于确定上下文的类型。

尽管上下文的名称可以设置为任意字符串，但使用它来模拟动态类型系统的最佳方法是将其直接设置为变量的类型。

建议使用 talloc（）和 talloc_array（）（或其变体）之一来创建上下文，因为它们会自动将上下文名称设置为给定类型的名称。

如果我们有一个名称之类的上下文，我们可以使用两个类似的函数来为我们执行类型检查和类型转换：

- talloc_get_type()
- talloc_get_type_abort()

### 3.2 示例

下面的例子将展示如何处理 talloc 的通用编程 —— 如果我们向回调提供无效数据，程序将被中止。在大多数应用中，这是对这种错误的充分反应。

```C
void foo_callback(void *pvt)
{
  struct foo *data = talloc_get_type_abort(pvt, struct foo);
  /* ... */
}

int do_foo()
{
  struct foo *data = talloc_zero(NULL, struct foo);
  /* ... */
  return generic_function(foo_callback, data);
}
```

但是，如果我们正在创建一个应该在服务器正常运行时间内运行的服务应用程序，我们可能希望在开发过程中中止该应用程序（以确保错误不会被忽略），并尝试从客户版本中的错误中恢复。这可以通过创建带有条件构建的自定义中止函数来实现。

```C
void my_abort(const char *reason)
{
  fprintf(stderr, "talloc abort: %s\n", reason);
#ifdef ABORT_ON_TYPE_MISMATCH
  abort();
#endif
}
```

talloc_get_type_abort() 的用法如下：

```C
talloc_set_abort_fn(my_abort);

TALLOC_CTX *ctx = talloc_new(NULL);
char *str = talloc_get_type_abort(ctx, char);
if (str == NULL) {
  /* recovery code */
}
/* talloc abort: ../src/main.c:25: Type mismatch:
   name[talloc_new: ../src/main.c:24] expected[char] */
```


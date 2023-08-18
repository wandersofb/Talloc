## 4 Using destructors

### 4.1 使用析构函数

析构函数是面向对象编程领域中众所周知的方法。析构函数是对象的一种方法，当对象被销毁时，它会自动运行。它通常用于将对象占用的资源返回系统（例如，关闭文件描述符、终止与数据库的连接、释放内存）。

有了 talloc，我们甚至可以在 C 中利用析构函数。我们可以很容易地将自己的析构函数附加到 talloc 上下文。当上下文被释放时，析构函数将自动运行。

要将析构函数附加/分离到 talloc 上下文，请使用：talloc_set_destructor()。

### 4.2 实例

想象一下，我们有一个动态创建的链表。在取消分配列表中的元素之前，我们需要确保已成功将其从列表中删除。通常，这将通过两个命令按照正确的顺序完成：将其从列表中删除，然后释放元素。使用 talloc，我们可以通过在元素上设置一个析构函数来立即实现这一点，析构函数将从列表中删除该元素，talloc_free（）将执行其余操作。

析构函数是：

```C
int list_remove(void *ctx)
{
    struct list_el *el = NULL;
    el = talloc_get_type_abort(ctx, struct list_el);
    /* remove element from the list */
}
```

GCC 版本 3 及更新版本可以在编译期间检查类型。因此，如果它是我们的主要编译器，我们可以使用更高级的析构函数：

```C
int list_remove(struct list_el *el)
{
    /* remove element from the list */
}
```

现在我们将析构函数分配给 list 元素。我们可以在插入它的函数中直接执行此操作。

```C
struct list_el* list_insert(TALLOC_CTX *mem_ctx,
                            struct list_el *where,
                            void *ptr)
{
  struct list_el *el = talloc(mem_ctx, struct list_el);
  el->data = ptr;
  /* insert into list */

  talloc_set_destructor(el, list_remove);
  return el;
}
```

因为 talloc 是一个分层内存分配器，所以我们可以更进一步，也可以使用元素释放数据：

```C
struct list_el* list_insert_free(TALLOC_CTX *mem_ctx,
                                 struct list_el *where,
                                 void *ptr)
{
  struct list_el *el = NULL;
  el = list_insert(mem_ctx, where, ptr);

  talloc_steal(el, ptr);

  return el;
}
```




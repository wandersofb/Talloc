## 5 Memory pools

### 5.1 Memory pools

分配新内存是一项昂贵的操作，大型程序可以为单个计算包含数千个 malloc() 调用，其中每个调用只分配非常少量的内存。这可能会导致应用程序的不希望的速度减慢。我们可以通过使用内存池来减少 malloc() 调用的数量，从而避免这种速度减慢。

内存池是预先分配的具有固定大小的内存空间。如果我们需要分配新的数据，我们将从池中获取所需的内存量，而不是从系统中请求新的内存。这是通过创建一个指向预分配内存内部的指针来完成的。这样的池不能被重新分配，因为它会改变位置 —— 指向池内的指针将无效。因此，内存池需要对所需内存空间进行非常好的估计。

talloc 库包含自己的内存池实现。它对程序员来说是高度透明的。唯一需要做的就是使用 talloc_pool() 初始化一个新的池上下文，它可以以与任何其他上下文相同的方式使用。

由于池上下文的以下属性，重构现有代码（使用 talloc）以利用内存池是非常简单的：
- 如果我们在池上下文上分配数据，它会从池中占用所需的内存量，
- 如果上下文是池上下文的后代，它也会占用池中的空间，
- 如果池没有足够的剩余内存，它将创建一个新的非池上下文，使池保持原样

```C
/* allocate 1KiB in a pool */
TALLOC_CTX *pool_ctx = talloc_pool(NULL, 1024);

/* Take 512B from the pool, 512B is left there */
void *ptr = talloc_size(pool_ctx, 512);

/* 1024B > 512B, this will create new talloc chunk outside
   the pool */
void *ptr2 = talloc_size(ptr, 1024);

/* The pool still contains 512 free bytes
 this will take 200B from them. */
void *ptr3 = talloc_size(ptr, 200);

/* This will destroy context 'ptr3' but the memory
 is not freed, the available space in the pool
 will increase to 512B. */
talloc_free(ptr3);

/* This will free memory taken by 'pool_ctx'
 and 'ptr2' as well. */
talloc_free(pool_ctx);
```

上面给出的非常方便，但有一个大问题需要记住。如果 talloc 池子级的父级更改为该池之外的父级，则在释放子级之前，不会释放整个池内存。因此，在窃取池上下文的后代时，我们必须非常小心。

```C
TALLOC_CTX *mem_ctx = talloc_new(NULL);
TALLOC_CTX *pool_ctx = talloc_pool(NULL, 1024);
struct foo *foo = talloc(pool_ctx, struct foo);

/* mem_ctx is not in the pool */
talloc_steal(mem_ctx, foo);

/* pool_ctx is marked as freed but the memory is not
   deallocated, accessing the pool_ctx again will cause
   an error */
talloc_free(pool_ctx);

/* This deallocates the pool_ctx. */
talloc_free(mem_ctx);
```

为了避免这个问题，通常最好复制我们想要的内存，而不是窃取它。如果我们不需要保留上下文名称（以保留类型信息），我们可以使用 talloc_memdup() 来做到这一点。

但是，根据复制内存的大小，从池中复制内存可能会放弃池提供的所有性能提升。因此，在采用此路径之前，应该对代码进行良好的分析。一般来说，黄金法则是：如果我们需要从池上下文中窃取，我们不应该使用池上下文。



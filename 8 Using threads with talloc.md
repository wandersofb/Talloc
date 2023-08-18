## 8 Using threads with talloc

### 8.1 线程安全

talloc 库在内部不是线程安全的，因为对 talloc 上下文上的变量的访问不受互斥或其他线程安全原语的控制。

但是，只要从主线程调用 talloc_disable_null_tracking() 来禁用 talloc 中的全局变量访问，那么每个线程都可以安全地使用从 null 上下文中分配的自己的顶级 talloc 上下文。
例如：

```C
static void *thread_fn(void *arg)
{
        const char *ctx_name = (const char *)arg;
        /*
 Create a new top level talloc hierarchy in
 this thread.
         */
        void *top_ctx = talloc_named_const(NULL, 0, "top");
        if (top_ctx == NULL) {
                return NULL;
        }
        sub_ctx = talloc_named_const(top_ctx, 100, ctx_name);
        if (sub_ctx == NULL) {
                return NULL;
        }

        /*
 Do more processing/talloc calls on top_ctx
 and its children.
         */
        ......

        talloc_free(top_ctx);
        return value;
}
```

是一种在线程中使用 talloc 的完全安全的方法。

当一个线程希望将在其本地顶级 talloc 上下文上分配的一些内存移动到另一个线程时，就会出现问题。必须小心添加数据访问排除，以防止内存损坏。一种方法是在对每个线程进行任何 talloc 调用之前锁定互斥体，但这会将 talloc 线程总体安全的负担推给库的糟糕用户。

在线程之间传输总计内存的一种更简单的方法是使用一个中间的、互斥锁锁定的中间变量。

下面是一个例子，取自 talloc 测试套件中的测试代码。

主线程创建 1000 个子线程，然后依次接受将一些线程分配的内存从每个线程转移到其顶级上下文。

pthread 互斥和条件变量用于通过 intermediate_ptr 变量同步传输。

```C
/* Required sync variables. */
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t condvar = PTHREAD_COND_INITIALIZER;

/* Intermediate talloc pointer for transfer. */
static void *intermediate_ptr;

/* Subthread. */
static void *thread_fn(void *arg)
{
        int ret;
        const char *ctx_name = (const char *)arg;
        void *sub_ctx = NULL;
        /*
 Do stuff that creates a new talloc hierarchy in
 this thread.
         */
        void *top_ctx = talloc_named_const(NULL, 0, "top");
        if (top_ctx == NULL) {
                return NULL;
        }
        sub_ctx = talloc_named_const(top_ctx, 100, ctx_name);
        if (sub_ctx == NULL) {
                return NULL;
        }

        /*
 Now transfer a pointer from our hierarchy
 onto the intermediate ptr.
         */
        ret = pthread_mutex_lock(&mtx);
        if (ret != 0) {
                talloc_free(top_ctx);
                return NULL;
        }

        /* Wait for intermediate_ptr to be free. */
        while (intermediate_ptr != NULL) {
                ret = pthread_cond_wait(&condvar, &mtx);
                if (ret != 0) {
                        talloc_free(top_ctx);
                        return NULL;
                }
        }

        /* and move our memory onto it from our toplevel hierarchy. */
        intermediate_ptr = talloc_move(NULL, &sub_ctx);

        /* Tell the main thread it's ready for pickup. */
        pthread_cond_broadcast(&condvar);
        pthread_mutex_unlock(&mtx);

        talloc_free(top_ctx);
        return NULL;
}

/* Main thread. */

#define NUM_THREADS 1000

static bool test_pthread_talloc_passing(void)
{
        int i;
        int ret;
        char str_array[NUM_THREADS][20];
        pthread_t thread_id;
        void *mem_ctx;

        /*
 Important ! Null tracking breaks threaded talloc.
 It *must* be turned off.
         */
        talloc_disable_null_tracking();

        /* Main thread toplevel context. */
        mem_ctx = talloc_named_const(NULL, 0, "toplevel");
        if (mem_ctx == NULL) {
                return false;
        }

        /*
 Spin off NUM_THREADS threads.
 They will use their own toplevel contexts.
         */
        for (i = 0; i < NUM_THREADS; i++) {
                (void)snprintf(str_array[i],
                                20,
                                "thread:%d",
                                i);
                if (str_array[i] == NULL) {
                        return false;
                }
                ret = pthread_create(&thread_id,
                                NULL,
                                thread_fn,
                                str_array[i]);
                if (ret != 0) {
                        return false;
                }
        }

        /* Now wait for NUM_THREADS transfers of the talloc'ed memory. */
        for (i = 0; i < NUM_THREADS; i++) {
                ret = pthread_mutex_lock(&mtx);
                if (ret != 0) {
                        talloc_free(mem_ctx);
                        return false;
                }

                /* Wait for intermediate_ptr to have our data. */
                while (intermediate_ptr == NULL) {
                        ret = pthread_cond_wait(&condvar, &mtx);
                        if (ret != 0) {
                                talloc_free(mem_ctx);
                                return false;
                        }
                }

                /* and move it onto our toplevel hierarchy. */
                (void)talloc_move(mem_ctx, &intermediate_ptr);

                /* Tell the sub-threads we're ready for another. */
                pthread_cond_broadcast(&condvar);
                pthread_mutex_unlock(&mtx);
        }

        /* Dump the hierarchy. */
        talloc_report(mem_ctx, stdout);
        talloc_free(mem_ctx);
        return true;
}
```
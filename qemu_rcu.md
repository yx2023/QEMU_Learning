## RCU原理
#### RCU原理 <br>
<a> https://zhuanlan.zhihu.com/p/89439043#:~:text=%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86,%E7%9C%8B%E6%88%96%E8%AE%B8%E4%BC%9A%E6%9B%B4%E5%8A%A0%E7%9B%B4%E8%A7%82%E3%80%82 </a><br>

#### qemu中RCU实现<br>
<a> https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2021/03/14/qemu-rcu </a><br>
<a> https://blog.csdn.net/woai110120130/article/details/100127123 </a><br>

RCU简单说就是rcu reader 通过 rcu_read_lock , rcu_read_unlock标定自己正在使用数据， 这段时间称作grace period<br>
rcu方式 write数据时， 会创建新数据（老数据copy), write作用在新数据上，并原子的用新数据替换老数据的指针的指向， <br>
如果发现有线程在 grace period中， 就不会释放老数据（数据副本），因为老数据正在被这些线程使用<br>
writer publish，判断当所有<b>使用老数据的rcu reader（线程）</b>都退出 grace period后， 老数据被free

<p style="color:red";>
QEMU publish是更新rcu_gp_ctr, 但这个rcu_gp_ctr在更新前，数据指针已经更新了，
所以持有老ctr的线程数据也可能是新的？

![image](https://user-images.githubusercontent.com/109275975/179889721-ef30ed48-6445-4264-a189-e6b4aa6cf916.png)

</p>
### 修改数据
实际上创建新数据，用新数据地址替换原指针指向，同时保留原数据，并将其（指向指针）推入队列dummy<br>
```
static void virtio_init_region_cache(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];
    VRingMemoryRegionCaches *old = vq->vring.caches;
    VRingMemoryRegionCaches *new = NULL;
  
    new = g_new0(VRingMemoryRegionCaches, 1);
    size = virtio_queue_get_desc_size(vdev, n);
    packed = virtio_vdev_has_feature(vq->vdev, VIRTIO_F_RING_PACKED) ?
                                   true : false;
    len = address_space_cache_init(&new->desc, vdev->dma_as,
                                   addr, size, packed);

    /* 
        qatomic_ruc_set -> __atmoic_store_n
        __atmoic_store_n(type *ptr, type val)
        将val放入目标指针
    */
    qatomic_rcu_set(&vq->vring.caches, new);
    /* 原子替换后其他线程再方问vq->vring.caches看到都是新数据 */
    if (old) {
    /* 老数据处理 */
        call_rcu(old, virtio_free_region_cache, rcu);
    }
    return;
}
```
### call_rcu
call_rcu -> call_ruc1 
1. 保存老数据的node进dummy队列
2. 计数器 +1
3. EV_SET，表明有rcu write 发生
```
/* func通常用于销毁（free）老数据 */
void call_rcu1(struct rcu_head *node, void (*func)(struct rcu_head *node))
{
    node->func = func;
    enqueue(node);
    qatomic_inc(&rcu_call_count);
    qemu_event_set(&rcu_call_ready_event);
}
```

### rcu_read_lock
1. 如果depth > 0，证明还在grace_period里，不更新ctr，退出
2. 用新的g_ctr更新local ctr
```
static inline void rcu_read_lock(void)
{
    /* 获取线程独有数据结构p_rcu_reader */
    struct rcu_reader_data *p_rcu_reader = get_ptr_rcu_reader();
    unsigned ctr;

    if (p_rcu_reader->depth++ > 0) {
        return;
    }

    ctr = qatomic_read(&rcu_gp_ctr);
    qatomic_set(&p_rcu_reader->ctr, ctr);

    /* 防止发生先获取旧数据。后拿到更新的g_ctr? */

    /* Write p_rcu_reader->ctr before reading RCU-protected pointers.  */
    smp_mb_placeholder();
}
```
### rcu_read_unlock
1. 如果depth > 0，证明还在grace_period里，不更新ctr，退出
2. 如果depth == 0，已经退出所有grace period，本地ctr为0，标记不持有任何旧数据
3. 如果wait_for_readers在等待你退出grace_period, waiting置为false, 发event唤醒他
```
static inline void rcu_read_unlock(void)
{
    struct rcu_reader_data *p_rcu_reader = get_ptr_rcu_reader();

    assert(p_rcu_reader->depth != 0);
    if (--p_rcu_reader->depth > 0) {
        return;
    }

    /* Ensure that the critical section is seen to precede the
     * store to p_rcu_reader->ctr.  Together with the following
     * smp_mb_placeholder(), this ensures writes to p_rcu_reader->ctr
     * are sequentially consistent.
     */
    qatomic_store_release(&p_rcu_reader->ctr, 0);

    /* 保证我做唤醒waiter之前，ctr状态已经对了 */

    /* Write p_rcu_reader->ctr before reading p_rcu_reader->waiting.  */
    smp_mb_placeholder();
    if (unlikely(qatomic_read(&p_rcu_reader->waiting))) {
        qatomic_set(&p_rcu_reader->waiting, false);
        qemu_event_set(&rcu_gp_event);
    }
}
```

### wait_for_readers
1. 遍历registry中所有thread的rcu_data, 置位rcu_data->waiting = true<br>
2. 如果thread[i]的rcu_data->ctr == g_ctr, 从registry移到qsreaders， 置位rcu_data->waiting = false<br>
3. 如果registry空了，goto 5<br>
4. 等待线程退出grace period事件 goto 1<br>
5. 所有rcu_data移回registry 退出函数<br>
```
static void wait_for_readers(void)
{
    ThreadList qsreaders = QLIST_HEAD_INITIALIZER(qsreaders);
    struct rcu_reader_data *index, *tmp;

    for (;;) {
        /* We want to be notified of changes made to rcu_gp_ongoing
         * while we walk the list.
         */
        qemu_event_reset(&rcu_gp_event);

        /* Instead of using qatomic_mb_set for index->waiting, and
         * qatomic_mb_read for index->ctr, memory barriers are placed
         * manually since writes to different threads are independent.
         * qemu_event_reset has acquire semantics, so no memory barrier
         * is needed here.
         */
        QLIST_FOREACH(index, &registry, node) {
            qatomic_set(&index->waiting, true);
        }

        /* Here, order the stores to index->waiting before the loads of
         * index->ctr.  Pairs with smp_mb_placeholder() in rcu_read_unlock(),
         * ensuring that the loads of index->ctr are sequentially consistent.
         */
        smp_mb_global();

        QLIST_FOREACH_SAFE(index, &registry, node, tmp) {
            if (!rcu_gp_ongoing(&index->ctr)) {
                QLIST_REMOVE(index, node);
                QLIST_INSERT_HEAD(&qsreaders, index, node);

                /* No need for mb_set here, worst of all we
                 * get some extra futex wakeups.
                 */
                qatomic_set(&index->waiting, false);
            } else if (qatomic_read(&in_drain_call_rcu)) {
                notifier_list_notify(&index->force_rcu, NULL);
            }
        }

        if (QLIST_EMPTY(&registry)) {
            break;
        }

        /* Wait for one thread to report a quiescent state and try again.
         * Release rcu_registry_lock, so rcu_(un)register_thread() doesn't
         * wait too much time.
         *
         * rcu_register_thread() may add nodes to &registry; it will not
         * wake up synchronize_rcu, but that is okay because at least another
         * thread must exit its RCU read-side critical section before
         * synchronize_rcu is done.  The next iteration of the loop will
         * move the new thread's rcu_reader from &registry to &qsreaders,
         * because rcu_gp_ongoing() will return false.
         *
         * rcu_unregister_thread() may remove nodes from &qsreaders instead
         * of &registry if it runs during qemu_event_wait.  That's okay;
         * the node then will not be added back to &registry by QLIST_SWAP
         * below.  The invariant is that the node is part of one list when
         * rcu_registry_lock is released.
         */
        qemu_mutex_unlock(&rcu_registry_lock);
        qemu_event_wait(&rcu_gp_event);
        qemu_mutex_lock(&rcu_registry_lock);
    }

    /* put back the reader list in the registry */
    QLIST_SWAP(&registry, &qsreaders, node);
}
```
### synchronize_rcu
1. 如果registry有线程， 更新rcu_gp_ctr
2. wait_for_readers
<b>QEMU_LOCK_GUARD</b> 
```
/**
 *
 *   ... <-- mutex not locked
 *   QEMU_LOCK_GUARD(&mutex); <-- mutex locked from here onwards
 *   ...
 *   if (error) {
 *       return; <-- mutex is automatically unlocked
 *   }
```

```
void synchronize_rcu(void)
{
    QEMU_LOCK_GUARD(&rcu_sync_lock);

    /* Write RCU-protected pointers before reading p_rcu_reader->ctr.
     * Pairs with smp_mb_placeholder() in rcu_read_lock().
     */
    smp_mb_global();

    /* 锁registry队列 */
    QEMU_LOCK_GUARD(&rcu_registry_lock);
    if (!QLIST_EMPTY(&registry)) {
        /* In either case, the qatomic_mb_set below blocks stores that free
         * old RCU-protected pointers.
         */
        if (sizeof(rcu_gp_ctr) < 8) {
            /* For architectures with 32-bit longs, a two-subphases algorithm
             * ensures we do not encounter overflow bugs.
             *
             * Switch parity: 0 -> 1, 1 -> 0.
             */
            qatomic_mb_set(&rcu_gp_ctr, rcu_gp_ctr ^ RCU_GP_CTR);
            wait_for_readers();
            qatomic_mb_set(&rcu_gp_ctr, rcu_gp_ctr ^ RCU_GP_CTR);
        } else {
            /* Increment current grace period.  */
            qatomic_mb_set(&rcu_gp_ctr, rcu_gp_ctr + RCU_GP_CTR);
        }

        wait_for_readers();
    }
}
```
### call_rcu_thread
1. 读rcu_call_count
2. 如果 n == 0 || (n < RCU_CALL_MIN_SIZE && ++tries <= 5 进3 否则 进6
3. 判断rcu_call_count是不是0, 0就休眠
4. 被call_rcu唤醒， 进2 
5. rcu_call_count减去本次处理的rcu write
6. synchronize_rcu
```
static void *call_rcu_thread(void *opaque)
{
    struct rcu_head *node;

    rcu_register_thread();

    for (;;) {
        int tries = 0;

        int n = qatomic_read(&rcu_call_count);

        /* Heuristically wait for a decent number of callbacks to pile up.
         * Fetch rcu_call_count now, we only must process elements that were
         * added before synchronize_rcu() starts.
         */

        while (n == 0 || (n < RCU_CALL_MIN_SIZE && ++tries <= 5)) {
            g_usleep(10000);
            if (n == 0) {

                /* flag置位EV_FREE */
                qemu_event_reset(&rcu_call_ready_event);
                n = qatomic_read(&rcu_call_count);
                if (n == 0) {
#if defined(CONFIG_MALLOC_TRIM)
                    malloc_trim(4 * 1024 * 1024);
#endif
                /* 
                    1. 如果ev->value是EV_FREE,置成EV_BUSY, cond_wait
                    2. 如果是EV_SET返回处理
                */ 
                    qemu_event_wait(&rcu_call_ready_event);
                }
            }
            n = qatomic_read(&rcu_call_count);
        }

        qatomic_sub(&rcu_call_count, n);
        synchronize_rcu();
        qemu_mutex_lock_iothread();
        while (n > 0) {
            node = try_dequeue();
            while (!node) {
                qemu_mutex_unlock_iothread();
                qemu_event_reset(&rcu_call_ready_event);
                node = try_dequeue();
                if (!node) {
                    qemu_event_wait(&rcu_call_ready_event);
                    node = try_dequeue();
                }
                qemu_mutex_lock_iothread();
            }

            n--;
            node->func(node);
        }
        qemu_mutex_unlock_iothread();
    }
    abort();
}
```
### builtin_function
<b>type __atomic_exchange_n (type *ptr, type val, int memorder)</b><br>
This built-in function implements an atomic exchange operation. It writes val into *ptr, and returns the previous contents of *ptr.<br>

<b>atomic_fetch_or( volatile A* obj, M arg )</b><br>
Atomically replaces the value pointed by obj with the result of bitwise OR between the old value of obj and arg, and returns the value obj held previously.<br>




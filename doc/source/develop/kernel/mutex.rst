.. _kernel_mutex:


同步-互斥量
###########

使用
====

API
---

**#define K_MUTEX_DEFINE(name)**

* 作用:定义一个k_mutex，并初始化
* name:k_mutex name

**int k_mutex_init(struct k_mutex *mutex)**

* 作用: 初始化mutex
* mutex: 要初始化的mutex
* 返回值: 0标示初始化成功

**int k_mutex_lock(struct k_mutex *mutex, s32_t timeout)**

* 作用：加锁
* mutex: 加锁的互斥量
* timeout: 等待时间，单位ms。K_NO_WAIT不等待,K_FOREVER一直等
* 返回值: 0表示加锁成功

**int k_mutex_unlock(struct k_mutex *mutex)**

* 作用：解锁
* mutex：解锁的互斥量
* 返回值: 0表示解锁成功，-EPERM表示当前thread并不拥有这个互斥量，-EINVAL表示该互斥量没有被锁

使用说明
--------

使用互斥量完成对资源的独占访问。 Mutex不能在ISR内使用。

初始化
~~~~~~

先初始化一个互斥量，下面两种方式的效果是一样的

方法1,使用宏

::

   K_MUTEX_DEFINE(my_mutex);

方法2，使用函数

::

   struct k_mutex my_mutex;
   k_mutex_init(&my_mutex);

资源独占访问
~~~~~~~~~~~~

下列示例代码中Thread
A和B都要去访问一个IO资源，但同一时间IO只能被独占访问，因此使用互斥量包含

::

   thread_A()
   {
       k_mutex_lock(&my_mutex, K_FOREVER);

       //Read IO
       ...

       k_mutex_unlock(&my_mutex);
   }

   thread_b()
   {
       k_mutex_lock(&my_mutex, K_FOREVER);

       //Write IO
       ...

       k_mutex_unlock(&my_mutex);
   }

实现
====

k_mutex结构体如下，可以看出其基本实现是用的wait_q

::

   struct k_mutex {
       /** Mutex wait queue */
       _wait_q_t wait_q;
       /** Mutex owner */
       struct k_thread *owner; //表示该mutex目前属于哪个线程

       /** Current lock count */
       u32_t lock_count;       //可重入锁用

       /** Original thread priority */
       int owner_orig_prio;    //优先级倒置用
   };


可重入锁是指如果一个线程已经拥有了互斥量，那么该线程可以继续多次对该互斥量加锁，同时也要做对应次数的解锁，才能完全释放该互斥量


初始化
------

k_mutex_init->z_impl_k_mutex_init，详细分析见注释

::

   int z_impl_k_mutex_init(struct k_mutex *mutex)
   {
       mutex->owner = NULL;        //全新的mutex是无owner的
       mutex->lock_count = 0U;     //次数也未加锁

       z_waitq_init(&mutex->wait_q);

       z_object_init(mutex);

       return 0;
   }

用宏也可以进行初始化

::

   #define _K_MUTEX_INITIALIZER(obj) \
       { \
       .wait_q = Z_WAIT_Q_INIT(&obj.wait_q), \     //等同于z_waitq_init
       .owner = NULL, \
       .lock_count = 0, \
       .owner_orig_prio = K_LOWEST_THREAD_PRIO, \
       _OBJECT_TRACING_INIT \
       }

   #define K_MUTEX_DEFINE(name) \
       Z_STRUCT_SECTION_ITERABLE(k_mutex, name) = \
           _K_MUTEX_INITIALIZER(name)

加锁
----

k_mutex_lock -> z_impl_k_mutex_unlock，会做下面几件事

1. 如果互斥量没其它线程用，直接获得互斥量返回

2. 如果互斥量是本线程在用，对可重入锁自加

3. 如果互斥锁被其它线程用了，进行优先级倒置调整，等待其它线程解锁互斥量

3. 如果超时内等到其它线程解锁互斥量，回去互斥量然后返回

4. 如果等互斥量超时，则放弃等待，检查是否有其它线程还在等待，已等待线程的优先级重新计算要倒置的优先级，重设拥有互斥量线程的优先级

::

   int z_impl_k_mutex_lock(struct k_mutex *mutex, s32_t timeout)
   {
       int new_prio;
       k_spinlock_key_t key;
       bool resched = false;

       key = k_spin_lock(&lock);


       //当前互斥量没被锁(lock_count ==0) 或是 当前thread已经拥有该锁(mutex->owner == _current)
       if (likely((mutex->lock_count == 0U) || (mutex->owner == _current))) {

           //记录thread当前的优先级，用于之后优先级倒置用
           mutex->owner_orig_prio = (mutex->lock_count == 0U) ?
                       _current->base.prio :
                       mutex->owner_orig_prio;

           mutex->lock_count++;        //对于未使用的锁这里lock_count会变成1，对于重入锁，这里lock_count会在原来的基础上增加然后返回
           mutex->owner = _current;    //更新owner

           k_spin_unlock(&lock, key);

           return 0;
       }

       //互斥量被其它thread占用，如果不等就立即返回
       if (unlikely(timeout == (s32_t)K_NO_WAIT)) {
           k_spin_unlock(&lock, key);
           return -EBUSY;
       }

       //如果要等，就进行判断，看自己线程的优先级和拥有互斥量的线程优先级谁高，计算一个新的优先级
       new_prio = new_prio_for_inheritance(_current->base.prio,
                           mutex->owner->base.prio);

       //如果互斥量拥有者线程的优先级比较低，则重设优先级，让优先级倒置
       if (z_is_prio_higher(new_prio, mutex->owner->base.prio)) {
           resched = adjust_owner_prio(mutex, new_prio);
       }

       //等待mutex释放，会引发调度
       int got_mutex = z_pend_curr(&lock, key, &mutex->wait_q, timeout);

       //等到mutex，返回
       if (got_mutex == 0) {
           return 0;
       }

       //等mutex超时
       key = k_spin_lock(&lock);

       //检查释放有其它线程在等待
       struct k_thread *waiter = z_waitq_head(&mutex->wait_q);

       //如果有其它线程在等待，比较Mutex拥有者线程和其它线程的优先级
       new_prio = (waiter != NULL) ?
           new_prio_for_inheritance(waiter->base.prio, mutex->owner_orig_prio) :
           mutex->owner_orig_prio;

       //重设拥有互斥量线程的优先级，并引发调度
       resched = adjust_owner_prio(mutex, new_prio) || resched;

       if (resched) {
           z_reschedule(&lock, key);
       } else {
           k_spin_unlock(&lock, key);
       }

       return -EAGAIN;
   }

解锁
----

k_mutex_unlock->z_impl_k_mutex_unlock，做下面几件事

1. 检查解锁者合法性
2. 接触重入锁
3. 恢复优先级倒置
4. 等待锁的线程获取mutex

::

   int z_impl_k_mutex_unlock(struct k_mutex *mutex)
   {
       struct k_thread *new_owner;

       //互斥量检查，不能解锁无owner的mutex
       CHECKIF(mutex->owner == NULL) {
           return -EINVAL;
       }

       //互斥量检查，不能解锁其它thread拥有的mutex
       CHECKIF(mutex->owner != _current) {
           return -EPERM;
       }

       //不允许解锁一个已经被完全
       __ASSERT_NO_MSG(mutex->lock_count > 0U);

       z_sched_lock();


       //可重入锁检查，如果没有全部解锁，直接退出
       if (mutex->lock_count - 1U != 0U) {
           mutex->lock_count--;
           goto k_mutex_unlock_return;
       }


       k_spinlock_key_t key = k_spin_lock(&lock);

       //mutex可重入锁已全部解完，对优先级倒置进行恢复
       adjust_owner_prio(mutex, mutex->owner_orig_prio);

       //检查释放有线程在等mutex
       new_owner = z_unpend_first_thread(&mutex->wait_q);

       mutex->owner = new_owner;

       if (new_owner != NULL) {
           //如果有线程在等mutex，该线程获取mutex并开始调度
           mutex->owner_orig_prio = new_owner->base.prio;
           arch_thread_return_value_set(new_owner, 0);
           z_ready_thread(new_owner);
           z_reschedule(&lock, key);
       } else {
           //如果没有线程等mutex，mutex空闲
           mutex->lock_count = 0U;
           k_spin_unlock(&lock, key);
       }


   k_mutex_unlock_return:
       k_sched_unlock();
       return 0;
   }

参考
====

https://docs.zephyrproject.org/latest/reference/kernel/synchronization/mutexes.html

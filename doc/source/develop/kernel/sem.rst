.. _kernel_sem:

同步-信号量
############

使用
====

API
---

**#define K_SEM_DEFINE(name, initial_count, count_limit)**

* 作用:声明一个k_sem，并初始化
* name: 声明一个name的k_sem
* initial_count:初始化count
* count_limit: 允许最大的count
* 无返回值，如果初始化有问题，会在编译期间出错

**int k_sem_init(struct k_sem *sem, unsigned int initial_count,unsigned int limit)**

* 作用: 初始化sem sem, 要初始化的sem
* initial_count:初始化
* count count_limit: 允许最大的count 返回值: 0表示初始化成功

**int k_sem_take(struct k_sem *sem, s32_t timeout)**

* 作用：获取信号
* sem: 要等待的信号量
* timeout: 等待时间，单位ms。K_NO_WAIT不等待,K_FOREVER一直等
* 返回值: 0表示拿到sem

**void k_sem_give(struct k_sem *sem)**

* 作用：发送信号

**void k_sem_reset(struct k_sem *sem)**

* 作用：将信号的量的count reset为0

**unsigned int k_sem_count_get(struct k_sem *sem)**

* 作用：获取指定信号量当前的count值

使用说明
--------

使用信号量来控制多个线程对一组资源的访问。
使用信号量在生产线程和消耗线程或ISR之间同步处理。

初始化
~~~~~~

先初始化一个信号量，下面两种方式的效果是一样的

方法1,使用宏

::

   K_SEM_DEFINE(my_sem, 0, 10);

方法2，使用函数

::

   struct k_sem my_sem;
   k_sem_init(&my_sem, 0, 10);

发送信号量
~~~~~~~~~~

允许在thread或者ISR中发送信号量，一般情况下发送信号量的thread或者ISR都是资源的生产者。
例如中断被触发时数据有效，在ISR中通过发信号量通知接收线程接收数据。

::

   void input_data_isr_handler(void *arg)
   {
       /* notify thread that data is available */
       k_sem_give(&my_sem);

       ...
   }

::

   void productor_thread(void *arg)
   {
       while(1){
           /* prepare data */
           ...
           /* notify thread that data is available */
           k_sem_give(&my_sem);
       }
   }

接收信号量
~~~~~~~~~~

允许在thread中接收信号量，但实际过程中ISR中接收信号量的情况几乎没有，如果一定要用，timeout只能用K_NO_WAIT。
一般情况下接收信号量的thread是消费者。

::

   void consumer_thread(void)
   {
       ...
       while(1){

           if (k_sem_take(&my_sem, K_MSEC(50)) != 0) {
               printk("Input data not available!");
           } else {
               /* fetch available data */
               ...
           }
       }
   }

实现
====

sem结构体如下,可以看出其基本实现是用的wait_q

::

   struct k_sem {
       _wait_q_t wait_q;
       u32_t count;        //记录当前sem cnt
       u32_t limit;        //最大cnt限制
       _POLL_EVENT;        //提供给poll用
   };

.. _初始化-1:

初始化
------

k_sem_init -> z_impl_k_sem_init ，流程分析见注释

::

   int z_impl_k_sem_init(struct k_sem *sem, unsigned int initial_count,
                 unsigned int limit)
   {
       //参数检测
       CHECKIF(limit == 0U || initial_count > limit) {
           return -EINVAL;
       }

       sem->count = initial_count;     //设置初始化的sem cnt
       sem->limit = limit;             //设置sem cnt最大限制
       z_waitq_init(&sem->wait_q);     //初始化wait_q
   #if defined(CONFIG_POLL)
       sys_dlist_init(&sem->poll_events);  //如果配置了poll，由于sem可以作为poll的条件，因此这里要初始化sem的poll_event
   #endif

       z_object_init(sem);

       return 0;
   }

再看一下使用宏初始化信号量的实现方法

::

   #define Z_SEM_INITIALIZER(obj, initial_count, count_limit) \
       { \
       .wait_q = Z_WAIT_Q_INIT(&obj.wait_q), \
       .count = initial_count, \
       .limit = count_limit, \
       _POLL_EVENT_OBJ_INIT(obj) \
       _OBJECT_TRACING_INIT \
       }

   #define K_SEM_DEFINE(name, initial_count, count_limit) \
       Z_STRUCT_SECTION_ITERABLE(k_sem, name) = \          定义一个k_sem变量
           Z_SEM_INITIALIZER(name, initial_count, count_limit); \   初始化这个变量
       BUILD_ASSERT(((count_limit) != 0) && \
                ((initial_count) <= (count_limit)));

发送
----

k_sem_give -> z_impl_k_sem_give，流程分析见注释

::

   void z_impl_k_sem_give(struct k_sem *sem)
   {
       k_spinlock_key_t key = k_spin_lock(&lock);

       //获取正在等待该sem的thread
       struct k_thread *thread = z_unpend_first_thread(&sem->wait_q);

       if (thread != NULL) {
           //如果存在等待sem的thread，将该thread转为就绪
           arch_thread_return_value_set(thread, 0);
           z_ready_thread(thread);
       } else {
           //如果不存在等待sem的thread, sem cnt +1, 将资源累计，但不能藏limit
           sem->count += (sem->count != sem->limit) ? 1U : 0U;

           //这里是通知poll该sem的对象条件已满足，这部分在poll分析
           handle_poll_events(sem);
       }

       //重新调度，切换ready的thread上
       z_reschedule(&lock, key);
   }

接收
----

k_sem_take->z_impl_k_sem_take，流程分析见注释

::

   int z_impl_k_sem_take(struct k_sem *sem, s32_t timeout)
   {
       int ret = 0;

       //ISR中只能不等待的收取sem
       __ASSERT(((arch_is_in_isr() == false) || (timeout == K_NO_WAIT)), "");

       k_spinlock_key_t key = k_spin_lock(&lock);

       //如果sem cnt不为0，可获取信号，直接返回
       if (likely(sem->count > 0U)) {
           sem->count--;
           k_spin_unlock(&lock, key);
           ret = 0;
           goto out;
       }

       //如果没有信号，且不愿意等待，直接返回
       if (timeout == K_NO_WAIT) {
           k_spin_unlock(&lock, key);
           ret = -EBUSY;
           goto out;
       }

       //没有信号，有要等待，会将等待的线程加入了等待列表中，然后重新调度切换到其它thread运行
       //等待超时或者等到sem后会从这里返回继续运行
       ret = z_pend_curr(&lock, key, &sem->wait_q, timeout);

   out:
       return ret;
   }

Reset
-----

k_sem_reset->z_impl_k_sem_reset 非常简单，将计数请0

::

   static inline void z_impl_k_sem_reset(struct k_sem *sem)
   {
       sem->count = 0U;
   }

Get cnt
-------

k_sem_count_get->z_impl_k_sem_count_get 也非常简单直接返回count

::

   static inline unsigned int z_impl_k_sem_count_get(struct k_sem *sem)
   {
       return sem->count;
   }

参考
====

https://docs.zephyrproject.org/latest/reference/kernel/synchronization/semaphores.html

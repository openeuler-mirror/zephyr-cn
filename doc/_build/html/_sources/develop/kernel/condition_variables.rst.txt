.. _kernel_condition_variables:

同步-条件变量
#############

条件变量通常用于控制共享资源的访问，它允许一个线程等待其它线程创建共享资源需要的条件。

使用
====

API
---

``#define K_CONDVAR_DEFINE(name)``

- 作用:定义一个k_condvar，并初始化

- name: k_condvar name

``int k_condvar_init(struct k_condvar *condvar)``

- 作用: 初始化条件变量

- condvar: 要初始化的condvar 返回值: 0标示初始化成功

``int k_condvar_signal(struct k_condvar *condvar)``

- 作用：通知条件变量有效，最先加入条件变量列队的thread将优先获得该有效条件

- condvar: 有效的条件变量 返回值: 0表示成功

``int k_condvar_broadcast(struct k_condvar *condvar)``

- 作用：广播条件变量有效，所有条件变量列队的thread将获得该有效条件

- condvar: 有效的条件变量 返回值: 0表示成功

``int k_condvar_wait(struct k_condvar *condvar, struct k_mutex *mutex, k_timeout_t timeout)``

- 作用：等待条件变量有效 condvar：等待的条件变量

- mutex：资源锁，条件变量和mutex搭配使用，在等待条件变量期间该mutex会被unlock

- timeout: 等待时间，单位ms。K_NO_WAIT不等待, K_FOREVER一直等

- 返回值: 0表示等待成功，-EAGAIN表示等待超时

使用说明
--------

初始化
~~~~~~

下面两种方法都可以定义&初始化一个条件变量 编译期初始化

::

   K_CONDVAR_DEFINE(my_condvar);

运行期初始化

::

   struct k_condvar my_condvar;
   k_condvar_init(&my_condvar);

等待条件有效
~~~~~~~~~~~~

::

   void thread_get(void)
   {
       k_mutex_lock(&mutex, K_FOREVER);

       //等待使用资源的条件有效，等待器件会解锁mutex，当等到后会重新上锁Mutex
       k_condvar_wait(&condvar, &mutex, K_FOREVER);

       //条件变量有效，使用共享资源
       ...
       k_mutex_unlock(&mutex);
   }

通知条件变量有效
~~~~~~~~~~~~~~~~

通知条件变量有效有两种方法：

通知一次，只有一个thread可以获得条件有效

::

   void worker_thread(void)
   {
       k_mutex_lock(&mutex, K_FOREVER);
       //通知条件有效
       k_condvar_signal(&condvar);
       k_mutex_unlock(&mutex);
   }

广播，所有等待该条件的thread都会获得条件有效

::

   void worker_thread(void)
   {
       k_mutex_lock(&mutex, K_FOREVER);
       //广播条件有效
       k_condvar_broadcast(&condvar);
       k_mutex_unlock(&mutex);
   }

使用条件变量需要搭配mutex来控制对共享资源的互斥访问。条件变量只是用来通知条件满足，条件变量不包含条件本身。

实现
====

struct k_condvar的结构体如下:

::

   struct k_condvar {
       _wait_q_t wait_q;
   };

条件变量就是由内核的wait_q实现的，条件变量的实现代码在kernel/condvar.c文件中，Zephyr中对内核对象的访问需要通过系统调用，Zephyr会将kernel.h中的API最后转化为调用系统调用的API例如:
k_condvar_init->z_impl_k_condvar_init
k_condvar_signal->z_impl_k_condvar_signal
k_condvar_broadcast->z_impl_k_condvar_broadcast
k_condvar_wait->z_impl_k_condvar_wait


初始化
------

代码非常简单，无论是使用那种方式初始化都是初始化一个wait_q

::

   int z_impl_k_condvar_init(struct k_condvar *condvar)
   {
       //初始化wait_q
       z_waitq_init(&condvar->wait_q);

       //初始化object，给userspace用，非userspace无作用
       z_object_init(condvar);

       SYS_PORT_TRACING_OBJ_INIT(k_condvar, condvar, 0);

       return 0;
   }

::

   #define Z_CONDVAR_INITIALIZER(obj)                                             \
       {                                                                      \
           .wait_q = Z_WAIT_Q_INIT(&obj.wait_q),                          \
       }

等待条件满足
------------

等待条件满足，就是将thread加入到条件变量的wait_q内等待，等待过程中会将资源锁放掉，当条件满足后又重新拿资源锁

::

   int z_impl_k_condvar_wait(struct k_condvar *condvar, struct k_mutex *mutex,
                 k_timeout_t timeout)
   {
       k_spinlock_key_t key;
       int ret;

       //锁调度，避免后面放资源锁的时候引发调度
       key = k_spin_lock(&lock);

       //释放资源锁
       k_mutex_unlock(mutex);

       //将当前线程加入到wait_q中，挂起当前线程，切换上下文
       ret = z_pend_curr(&lock, key, &condvar->wait_q, timeout);

       //条件变量满足后，会从wait_q取出挂起的线程从这里恢复执行

       //重新拿到资源锁
       k_mutex_lock(mutex, K_FOREVER);


       return ret;
   }

通知条件满足
------------

通知条件满足会从条件变量的wait_q中取出一个等待的thread恢复执行，实现分析如下

::

   int z_impl_k_condvar_signal(struct k_condvar *condvar)
   {
       //锁调度
       k_spinlock_key_t key = k_spin_lock(&lock);

       //从条件变量的wait_q中选出最合适的thread
       struct k_thread *thread = z_unpend_first_thread(&condvar->wait_q);

       if (thread != NULL) {
           //如果存在等待条件变量的thread

           //设置返回为0
           arch_thread_return_value_set(thread, 0);

           //将其转为就绪状态
           z_ready_thread(thread);

           //引发重新调度
           z_reschedule(&lock, key);
       } else {
           //如果不存在等待条件变量的thread，解锁调度，退出
           k_spin_unlock(&lock, key);
       }
       return 0;
   }

广播条件满足
------------

广播条件满足会从条件变量的wait_q中取出所有的thread恢复执行，实现分析如下

::

   int z_impl_k_condvar_broadcast(struct k_condvar *condvar)
   {
       struct k_thread *pending_thread;
       k_spinlock_key_t key;
       int woken = 0;

       //锁调度
       key = k_spin_lock(&lock);


       //从wait_q中取出所有thread进行恢复
       while ((pending_thread = z_unpend_first_thread(&condvar->wait_q)) !=
              NULL) {
           //设置返回为0
           arch_thread_return_value_set(pending_thread, 0);
           //将其转为就绪状态
           z_ready_thread(pending_thread);
       }

       //引发调度
       z_reschedule(&lock, key);

       return woken;
   }

和信号量的区别
==============

初看条件变量总有一点信号量的影子，Zephyr中二者主要有如下区别:

1. 信号量无法进行广播。

2. 多值信号量可以积压，没有消费者发送的信号依然被保存，而条件变量发送后没有消费者接受条件就过期了。

3. 条件变量必须搭配互斥锁使用。

4. Zephyr中信号量可以用于Poll，条件变量则不行。

参考
====

https://docs.zephyrproject.org/latest/reference/kernel/synchronization/condvar.html

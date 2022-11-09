.. _kernel_event:

同步-事件
##########

事件对象是一种线程同步对象，允许一个或者多个线程等待同一个事件对象，当事件被传递到事件对象时，满足条件的线程都变为就绪。通常使用事件来通知一系列条件已满足。在Zephyr中每个事件对象使用一个32bit数来跟踪传递的事件集，每个bit表示一个事件。事件可以由ISR或者线程发送，发送事件有两种方法：

* 设置：覆盖现有的事件集，重写这个32bit数

* 发布：以bit方式\ **添加**\ 事件到事件基，注意这里只能添加，不能删除

线程可以等待一个或多个事件。在等待多个事件时可以指定等待其中部分或者全部事件。线程在提出等待事件对象时，可以选择在等待前清空等待事件对象上的所有事件。


API
===

宏定义
------

**K_EVENT_DEFINE(name)**

- 静态的定义和初始化事件对象。

-  **name**  事件对象名

函数
----

**void k_event_init(struct k_event *event)**

- 初始化事件对象，在使用事件对象前首先使用该函数对其进行初始化。

- **event** 事件对象的地址

**void k_event_post(struct k_event *event, uint32_t events)**

- 发布一个或多个事件到事件对象。如果等待\ ``event``\ 的线程因为发布的事件满足等待条件，该线程被转为就绪，与设置事件不同，发布的事件是增加到事件对象中。

-  **event**  事件对象的地址

-  **events**  发布的事件集合

**void k_event_set(struct k_event *event, uint32_t events)**

设置事件集合到事件对象，将事件对象中的事件设置为\ ``events``\ 。如果等待\ ``event``\ 的线程因为设置的事件满足等待条件，该线程被转为就绪。与发布事件不同，设置事件会取代事件对象中的事件集。

-  **event**  事件对象的地址
-  **events**  设置的事件集合

**uint32_t k_event_wait(struct k_event *event, uint32_t events, bool reset, k_timeout_t timeout)**

等待任意指定事件，在\ ``event``\ 上等待，有任意指定的事件被发送，或者是等待时间超过\ ``timeout``\ 。一个线程最多可以等待32个事件，这些事件由\ ``events``\ 中的bit位置标识。

**注意**
``reset = true``\ 将在等待事件前，将\ ``event``\ 中已发生的事件，使用的时候请特别注意。

- **event**  事件对象的地址
- **events**  等待的事件集
- **reset** 如果为\ ``true``\ ，等待前清空\ ``event``\ 中已发生的事件，如果为false则不清空
- **timeout**  指定等待事件的最长事件，可以指定为\ ``K_NO_WAIT`` 和\ ``K_FOREVER``

- 返回值  **事件集**  等待成功后返回收到匹配的事件集 **0** 指定时间内未收到匹配事件

**uint32_t k_event_wait_all(struct k_event *event, uint32_t events, bool reset, k_timeout_t  timeout)**

等待所有指定事件，在\ ``event``\ 上等待，指定的所有事件被发送，或者是等待时间超过\ ``timeout``\ 。一个线程最多可以等待32个事件，这些事件由\ ``events``\ 中的bit位置标识。

**注意**
``reset = true``\ 将在等待事件前，将\ ``event``\ 中已发生的事件，使用的时候请特别注意。

- **event**  事件对象的地址
- **events**  等待的事件集
- **reset** 如果为\ ``true``\ ，等待前清空\ ``event``\ 中已发生的事件，如果为false则不清空
- **timeout** 指定等待事件的最长事件，可以指定为\ ``K_NO_WAIT`` 和 ``K_FOREVER``

返回值

-  **事件集**  等待成功后返回收到匹配的事件集
-  **0**  指定时间内未收到匹配事件

使用
====

配置
----

事件对象的配置选项是\ ``CONFIG_EVENTS``\ ，zephyr内核没有给事件对象配置选项设置默认值，编译时将被识别为未配置，因此要使用事件对象需要增加配置\ ``CONFIG_EVENTS=y``\ 。

示例
----

**初始化事件对象**

使用函数

.. code:: c

   struct k_event my_event;
   k_event_init(&my_event);

等效于

.. code:: c

   K_EVENT_DEFINE(my_event);

**设置事件**

下列示例在中断服务程序中设置事件为0x001

.. code:: c

   void input_available_interrupt_handler(void *arg)
   {
       /* 通知线程数据有效 */
       k_event_set(&my_event, 0x001);

       ...
   }

**发布事件**

下面示例在中断服务程序中发布事件0x120，如果前面的设置事件未被清除，此时\ ``my_event``\ 中的事件为0x121

.. code:: c

   void input_available_interrupt_handler(void *arg)
   {
       ...

       /* notify threads that more data is available */

       k_event_post(&my_event, 0x120);

       ...
   }

**等待事件**

下面示例等待事件50毫秒，在50毫秒内只要前面的设置和发布示例中任意一个发生都会等待成功

.. code:: c

   void consumer_thread(void)
   {
       uint32_t  events;

       events = k_event_wait(&my_event, 0xFFF, false, K_MSEC(50));
       if (events == 0) {
           printk("No input devices are available!");
       } else {
           /* 收到事件，根据events进行处理 */
           ...
       }
       ...
   }

下面示例等待事件50毫秒，在50毫秒没只有前面的设置和发布示例都发生了才会等待成功

.. code:: c

   void consumer_thread(void)
   {
       uint32_t  events;

       events = k_event_wait_all(&my_event, 0x121, false, K_MSEC(50));
       if (events == 0) {
           printk("At least one input device is not available!");
       } else {
           /* 事件全部收齐，进行处理 */
           ...
       }
       ...
   }

代码分析
========

.. raw:: html

   <!--Zephyr version: 2.7.99 (/mnt/g/zpro/zephyr), build: e8df8a579033-->

事件对象的代码实现在\ ``kernel\events.c``\ ，事件对象是由\ ``struct k_event``\ 进行管理

.. code:: c

   struct k_event {
       _wait_q_t         wait_q;
       uint32_t          events;
       struct k_spinlock lock;
   };

-  ``wait_q`` 用于管理等待该事件对象的线程

-  ``events`` 用于保存当前事件对象收到的事件

-  ``lock`` 用于保护内核对事件对象的操作的原子性

事件对象的所有操作都是围绕着\ ``struct k_event``\ 进行的。

初始化
------

函数实现代码如下，就是对\ ``struct k_event``\ 定义事件的各个成员进行初始化

.. code:: c

   void z_impl_k_event_init(struct k_event *event)
   {
       event->events = 0;
       event->lock = (struct k_spinlock) {};

       z_waitq_init(&event->wait_q);

       z_object_init(event);
   }

用宏可以达到同时定义和初始化的目的，实现如下

.. code:: c

   #define Z_EVENT_INITIALIZER(obj) \
       { \
       .wait_q = Z_WAIT_Q_INIT(&obj.wait_q), \
       .events = 0 \
       }

   #define K_EVENT_DEFINE(name)                               \
       STRUCT_SECTION_ITERABLE(k_event, name) =               \
           Z_EVENT_INITIALIZER(name);

等待事件
--------

等待事件可以等待任意指定事件和全部事件，在\ ``events.c``\ 中都是由同一个内部函数\ ``k_event_wait_internal``\ 实现，只是指定的参数不一样

.. code:: c

   uint32_t z_impl_k_event_wait(struct k_event *event, uint32_t events,
                    bool reset, k_timeout_t timeout)
   {
       uint32_t options = reset ? K_EVENT_WAIT_RESET : 0;

       return k_event_wait_internal(event, events, options, timeout);
   }

   uint32_t z_impl_k_event_wait_all(struct k_event *event, uint32_t events,
                    bool reset, k_timeout_t timeout)
   {
       /* 使用K_EVENT_WAIT_ALL选项，表示要等待所有的事件收齐 */
       uint32_t options = reset ? (K_EVENT_WAIT_RESET | K_EVENT_WAIT_ALL)
                    : K_EVENT_WAIT_ALL;

       return k_event_wait_internal(event, events, options, timeout);
   }

内部函数\ ``k_event_wait_internal``\ 的第三个参数\ ``opetions``\ 用于指定等待的选项，选项定义在\ ``events.c``\ 中

.. code:: c

   #define K_EVENT_WAIT_ANY      0x00   /* 有1个或者以上事件满足就可退出等待 */
   #define K_EVENT_WAIT_ALL      0x01   /* 所有事件满足才可退出等待 */
   #define K_EVENT_WAIT_RESET    0x02   /* 等待事件前先清空已有事件 */

   #define K_EVENT_WAIT_MASK     0x01    /* 用于获取等待类型 */

.. code:: c

   static uint32_t k_event_wait_internal(struct k_event *event, uint32_t events,
                         unsigned int options, k_timeout_t timeout)
   {
       uint32_t  rv = 0;
       unsigned int  wait_condition;
       struct k_thread  *thread;

       /* isr中只能不做任何等待的等待事件 */
       __ASSERT(((arch_is_in_isr() == false) ||
             K_TIMEOUT_EQ(timeout, K_NO_WAIT)), "");

       /* 不允许等待的事件集为0，相当于未等待任何事件 */
       if (events == 0) {
           return 0;
       }

       wait_condition = options & K_EVENT_WAIT_MASK;
       thread = z_current_get();

       k_spinlock_key_t  key = k_spin_lock(&event->lock);

       /* 检查是否需要清空已有事件 */
       if (options & K_EVENT_WAIT_RESET) {
           event->events = 0;
       }

       /* 检查事件对象已有的事件是否已经满足线程，如果满足则退出 */
       if (are_wait_conditions_met(events, event->events, wait_condition)) {
           rv = event->events;

           k_spin_unlock(&event->lock, key);
           goto out;
       }

       /* 如果等待的超时未立即退出，则不进行等待，此时rv=0 */
       if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
           k_spin_unlock(&event->lock, key);
           goto out;
       }

       /* 将线程要等待的事件集和方式保存到线程中 */
       thread->events = events;
       thread->event_options = options;

       /* 等待事件发生，如果等待超时rv仍然保持为0 */
       if (z_pend_curr(&event->lock, key, &event->wait_q, timeout) == 0) {
           /* 等待事件已发生，发送事件者将把满足的事件交换到线程内的events中，
              rv中保存了等待到的事件
            */
           rv = thread->events;
       }

   out:
       /* 由于发生的事件可能会超出等待的事件，因此需要做位与返回 */
       return rv & events;
   }

发送事件
--------

发送事件分为设置和发布，在\ ``events.c``\ 中都是由同一个内部函数\ ``k_event_post_internal``\ 实现，只是指定的参数不一样

.. code:: c

   void z_impl_k_event_post(struct k_event *event, uint32_t events)
   {
       k_event_post_internal(event, events, true);
   }

   void z_impl_k_event_set(struct k_event *event, uint32_t events)
   {
       k_event_post_internal(event, events, false);
   }

内部函数\ ``k_event_post_internal``\ 的第三个参数\ ``accumulat``\ 为\ ``true``\ 时表示发送的\ ``events``\ 是添加到\ ``event``\ 内，为\ ``false``\ 时表示是覆盖\ ``event``\ 的已有事件

.. code:: c

   static void k_event_post_internal(struct k_event *event, uint32_t events,
                     bool accumulate)
   {
       k_spinlock_key_t  key;
       struct k_thread  *thread;
       unsigned int      wait_condition;
       struct k_thread  *head = NULL;

       /* 上锁，保证内核对事件对象操作的原子性 */
       key = k_spin_lock(&event->lock);

       /* 检查是附加事件，还是要重置事件 */
       if (accumulate) {
           /* 附加事件 */
           events |= event->events;
       }
       /* 对发生的事件进行更新 */
       event->events = events;

       /* 遍历事件对象中wait_q内存放等待的thread，并将事件能满足的线程加入到单链表中 */
       _WAIT_Q_FOR_EACH(&event->wait_q, thread) {
           /* 获取等待类型K_EVENT_WAIT_MASK为0x01，只取最低位，
              这里wait_condition是K_EVENT_WAIT_ANY或K_EVENT_WAIT_ALL
            */
           wait_condition = thread->event_options & K_EVENT_WAIT_MASK;

           /* 根据等待类型对线程进行事件匹配，匹配上的线程放入单链表head中 */
           if (are_wait_conditions_met(thread->events, events,
                           wait_condition)) {
               thread->next_event_link = head;
               head = thread;
           }
       }

       /* 对事件匹配上的线程单链表进行遍历，通知线程就绪 */
       if (head != NULL) {
           thread = head;
           do {
               z_unpend_thread(thread);
               arch_thread_return_value_set(thread, 0);
               /* 更新线程收到的事件 */
               thread->events = events;
               /* 线程恢复就绪 */
               z_ready_thread(thread);
               thread = thread->next_event_link;
           } while (thread != NULL);
       }
       /* 发送完事件后，引发调度，让刚变为就绪线程有机会执行 */
       z_reschedule(&event->lock, key);
   }

参考
====

https://docs.zephyrproject.org/latest/reference/kernel/synchronization/events.html

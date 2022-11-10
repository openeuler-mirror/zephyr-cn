.. _kernel_mailbox:

数据传递-邮箱
###############

Zephyr的邮箱(mailbox)可以看作是升级版的消息队列(msgq)，与msgq不一样的是：邮箱的消息可以指定发送者和接收者，邮箱消息的大小可以不固定，长度也没有对齐的要求。此外邮箱只能用于线程间交换消息，不能用于ISR内。

邮箱构成和特性
===============

Zephyr允许定义任意数量的邮箱，最大数量只由可用内存的量决定。
一个邮箱主要是由发送队列和消息队列两个wait_q组才，如下定义：

::

   struct k_mbox {
       _wait_q_t tx_msg_queue;
       _wait_q_t rx_msg_queue;
       struct k_spinlock lock;
   };

-  发送队列：用于保存已发送但还未被接收的消息
-  接收队列：用于保存正在等待消息的线程

当发送线程发送消息到邮箱后会阻塞等待(同步发送)接收线程完成消息接收才退出等待，或是不阻塞而是等待接收线程完成消息接收后通过信号量通知(异步发送)。
邮箱支持：

* 一对一： A发到邮箱的消息只能B收, B只收A的消息
* 一对多： A发到邮箱的消息任意线程都可以收，注意不是广播。
* 多对一： B线程可以收任意线程发到邮箱中消息。

邮箱不支持广播，邮箱中的消息只能被一个线程使用。

消息
----

消息描述符
~~~~~~~~~~

消息是由消息描述符描述，指定消息的内存地址，并描述消息如何被邮箱处理。发送线程和接收线程在访问邮箱时都需要提供消息描述符，消息描述符相互匹配才能传递消息。
消息描述符的结构如下：

::

   struct k_mbox_msg {
       uint32_t _mailbox;
       size_t size;
       uint32_t info;
       void *tx_data;
       void *_rx_data;
       struct k_mem_block tx_block;
       k_tid_t rx_source_thread;
       k_tid_t tx_target_thread;
       k_tid_t _syncing_thread;
   #if (CONFIG_NUM_MBOX_ASYNC_MSGS > 0)
       struct k_sem *_async_sem;
   #endif
   };

``_mailbox``\ 和\ ``_rx_data``\ 这两个字段已经不再使用
``_syncing_thread``
用于通知发送者发送的消息已经被接收，\ ``_async_sem``\ 用于异步发送后通知消息已经完成，这两个字段都是内部使用。

消息描述符中容纳消息的方式有两种：

* 缓冲：给定一段buffer，消息数据放于其中
* 内存块：申请的一片内存块，消息数据放于其中

实际的消息发送二选一，如果二者都为NULL，则表示发送空消息。

``tx_data`` 发送缓冲消息的内存指针，如果是内存块消息或者空消息该字段为NULL，接收者不初始化该字段
``tx_block`` 内存块id，发送内存块消息时使用该字段。如果是缓冲消息或是空消息该字段为NULL，接收者不初始化该字段。
``size`` 消息大小，允许为0，为0时表示空消息. 接收时表示允许接收消息的最大size,如果收到的消息不是想要的发送方的size将为设为0，如果消息被接收，发送方的size将被更新为实际接收的大小。
``info`` 发送者和接收者在消息接收时双向交换字段，32bit大小
``tx_target_thread`` 接收消息的线程id，接收者消息不初始化该字段，接收者接收该消息后邮箱会将该字段更新为为实际接收者的id。当指定为K_ANY时该消息可以被任意线程接收。
``rx_source_thread`` 发送消息的线程id，发送者消息不初始化该字段，接收者接收该消息后邮箱会将该字段更新为为实际发送者的id。当指定为K_ANY时该线程可接收任意消息。

消息的生命周期
~~~~~~~~~~~~~~

创建消息：发送线程将消息发送到邮箱 消息保存在邮箱
删除消息：接收线程从邮箱接收消息，并将其消耗

使用
====

API
---

``#define K_MBOX_DEFINE(name)``

作用:定义一个k_mbox，并初始化

name: k_mbox name

``void k_mbox_init(struct k_mbox *mbox)``

作用: 初始化k_mbox

mbox: 要初始化的mbox

返回值: 0标示初始化成功

``int k_mbox_put(struct k_mbox *mbox, struct k_mbox_msg *tx_msg, k_timeout_t timeout)``
作用：同步发送消息到邮箱，会等到有线程从邮箱接收，或是超时才退出。如果发送超时消息将不会被保存在邮箱中

mbox: 邮箱 tx_msg: 发送消息的描述符，包含了接收者，消息信息，见前文说明

timeout： 等待时间，单位ms。K_NO_WAIT不等待, K_FOREVER一直等

返回值：0发送成果，-ENOMSG不等待发送失败，-EAGAIN 等待超时

``void k_mbox_async_put(struct k_mbox *mbox, struct k_mbox_msg *tx_msg, struct k_sem *sem)``

作用：异步发送消息到邮箱，发送后立即返回，有线程从邮箱接收该消息时使用sem通知。

mbox: 邮箱 tx_msg: 发送消息的描述符，包含了接收者，消息信息，见前文说明

sem：接收消息通知信号量

``int k_mbox_get(struct k_mbox *mbox, struct k_mbox_msg *rx_msg, void *buffer, k_timeout_t timeout)``

作用：从邮箱接收消息。

mbox: 邮箱

rx_msg: 接收消息的描述符，指定发送者，接收消息的信息，见前文说明

buffer: 接收消息数据存放buffer，如果为NULL，邮箱将保存该消息，消息的信息保存在rx_msg中，直到使用k_mbox_data_get消耗该消息才会从邮箱中删除

timeout： 等待时间，单位ms。K_NO_WAIT不等待, K_FOREVER一直等

返回值：0发送成果，-ENOMSG不等待发送失败，-EAGAIN 等待超时

``void k_mbox_data_get(struct k_mbox_msg *rx_msg, void *buffer)``

作用：将消息数据搬运到buffer中，并从邮箱中删除该消息。

rx_msg: 接收消息的描述符，包含了要接收消息的信息

buffer: 接收消息数据存放buffer，如果为NULL将直接丢弃该消息

使用说明
--------

初始化
~~~~~~

先定义初始化一个邮箱，下面两种方式的效果是一样的 使用宏

::

   K_MBOX_DEFINE(my_mailbox);

使用函数

::

   struct k_mbox my_mailbox;
   k_mbox_init(&my_mailbox);

发送消息
~~~~~~~~

同步

::

   void producer_thread(void)
   {
       char buffer[100];
       int buffer_bytes_used;

       struct k_mbox_msg send_msg;

       while (1) {

           //准备要发送的消息数据
           ...
           buffer_bytes_used = ... ;
           memcpy(buffer, source, buffer_bytes_used);

           //准备消息描述符
           send_msg.info = 123;
           send_msg.size = buffer_bytes_used;
           send_msg.tx_data = buffer;
           send_msg.tx_block.data = NULL;
           send_msg.tx_target_thread = consumer_thread_id;

           //发送消息到mailbox，并等待被接收
           k_mbox_put(&my_mailbox, &send_msg, K_FOREVER);

           //接收完毕退出等待，检查info, size, tx_target_thread 的更新
           //info和接收者交换会变为456
           //size变为实际接收的30

           /* verify that message data was fully received */
           if (send_msg.size < buffer_bytes_used) {
               printf("some message data dropped during transfer!");
               printf("receiver only had room for %d bytes", send_msg.info);
           }
       }
   }

异步，异步发送函数\ ``k_mbox_async_put``\ 在配置了\ ``CONFIG_NUM_MBOX_ASYNC_MSGS=y``\ 后才会有效

::

   void producer_thread(void)
   {
       char buffer[100];
       int buffer_bytes_used = 100;

       struct k_mbox_msg send_msg;

       struct k_sem rev_sem;
       k_sem_init(&rev_sem, 0, 10);

       while (1) {

           //准备消息描述符
           ...
           buffer_bytes_used = ... ;
           memcpy(buffer, source, buffer_bytes_used);

           /* prepare to send message */
           send_msg.info = 123;
           send_msg.size = buffer_bytes_used;
           send_msg.tx_data = buffer;
           send_msg.tx_block.data = NULL;
           send_msg.tx_target_thread = consumer_thread_id;

           //发送消息到mailbox
           k_mbox_async_put(&my_mailbox, &send_msg, &rev_sem);

           //等待消息被接收通知信号量
           k_sem_take(&rev_sem, K_FOREVER);

           //接收完毕退出等待，检查info, size, tx_target_thread 的更新
           //info和接收者交换会变为456
           //size变为实际接收的30

           /* verify that message data was fully received */
           if (send_msg.size < buffer_bytes_used) {
               printf("some message data dropped during transfer!");
               printf("receiver only had room for %d bytes", send_msg.info);
           }
       }
   }

接收消息
~~~~~~~~

::

   void consumer_thread(void)
   {
       struct k_mbox_msg recv_msg;
       char buffer[100];

       int i;
       int total;

       while (1) {
           //准备消息描述符
           recv_msg.info = 456;
           recv_msg.size = 30;
           recv_msg.rx_source_thread = producer_thread_id;

           //等待接收消息
           k_mbox_get(&my_mailbox, &recv_msg, buffer, K_FOREVER);

           //接收完毕退出等待，检查info, size, rx_target_thread 的更新
           //info和发收者交换会变为123
           //size为实际接收的数据长度30

           /* verify that message data was fully received */
           if (recv_msg.info != recv_msg.size) {
               printf("some message data dropped during transfer!");
               printf("sender tried to send %d bytes", recv_msg.info);
           }

           /* compute sum of all message bytes (from 0 to 100 of them) */
           total = 0;
           for (i = 0; i < recv_msg.size; i++) {
               total += buffer[i];
           }
       }
   }

实现
====

该小节通过对邮箱内核代码的分析，理解Zephyr是如何实现以上描述的功能特性.
mailbox的实现代码在kernel:raw-latex:`\mailbox`.c中 ## 初始化
前文提到过mailbox的核心就是两个wait_q，初始化则是对发送和接收的wait_q进行初始话，并初始化一个lock用于操作wait_q时锁调度

::

   void k_mbox_init(struct k_mbox *mbox)
   {
       z_waitq_init(&mbox->tx_msg_queue);
       z_waitq_init(&mbox->rx_msg_queue);
       mbox->lock = (struct k_spinlock) {};
   }


发送消息
--------

这里主要分析同步发送，\ ``k_mbox_put->mbox_message_put``

::

   int k_mbox_put(struct k_mbox *mbox, struct k_mbox_msg *tx_msg,
              k_timeout_t timeout)
   {
       //发送消息的线程在消息被接收前会被放入tx_msg_queue中挂起，这里将当前thread保存在消息中的_syncing_thread，方便消息被接收后对_syncing_thread进行恢复
       tx_msg->_syncing_thread = _current;

       int ret = mbox_message_put(mbox, tx_msg, timeout);

       return ret;
   }

消息发送的主要流程位于\ ``mbox_message_put``\ 中，有下面两种情况：

1. 遍历邮箱\ ``rx_msg_queue``\ 中等待消息的线程，如果找到匹配的接收线程，让接收线程变为就绪，挂起当前线程。
2. 如果没找到匹配的接收线程，将消息放入\ ``tx_msg_queue``\ ，挂起当前线程并等待超时。

代码详细分析如下：

::

   static int mbox_message_put(struct k_mbox *mbox, struct k_mbox_msg *tx_msg,
                    k_timeout_t timeout)
   {
       struct k_thread *sending_thread;
       struct k_thread *receiving_thread;
       struct k_mbox_msg *rx_msg;
       k_spinlock_key_t key;

       //设置描述符中发送线程id为当前线程
       tx_msg->rx_source_thread = _current;

       //发送线程要交换的数据被设置为消息描述符
       sending_thread = tx_msg->_syncing_thread;
       sending_thread->base.swap_data = tx_msg;

       //锁调度，开始处理消息发送
       key = k_spin_lock(&mbox->lock);

       //检查rx_msg_queue中是否有线程在等待接收邮箱消息
       _WAIT_Q_FOR_EACH(&mbox->rx_msg_queue, receiving_thread) {
           //获取接收消息描述符
           rx_msg = (struct k_mbox_msg *)receiving_thread->base.swap_data;

           //将收发消息描述进行匹配
           if (mbox_message_match(tx_msg, rx_msg) == 0) {
               //该消息有线程可匹配，让接收线程从挂起恢复
               z_unpend_thread(receiving_thread);

               //pend返回设置为0，表示有正常拿到消息
               arch_thread_return_value_set(receiving_thread, 0);
               //让接收线程处于就绪态，下一次调度的时候接收线程就会取得消息进行处理
               z_ready_thread(receiving_thread);

   #if (CONFIG_NUM_MBOX_ASYNC_MSGS > 0)
               //异步发送流程，接收线程处理消失时不做阻塞
               if ((sending_thread->base.thread_state & _THREAD_DUMMY)
                   != 0U) {
                   z_reschedule(&mbox->lock, key);
                   return 0;
               }
   #endif

               //同步发送流程，异步线程处理消息时，发送消息线程被直接挂起，异步线程处理完后会通过线程交换数据中保存的tx_msg->_syncing_thread让发送线程进入就绪继续运行
               int ret = z_pend_curr(&mbox->lock, key, NULL, K_FOREVER);

               return ret;
           }
       }

       //没有匹配的接收线程，同时也不等待消息发送，就立即退出
       if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
           k_spin_unlock(&mbox->lock, key);
           return -ENOMSG;
       }

   #if (CONFIG_NUM_MBOX_ASYNC_MSGS > 0)
       //异步发送时会用一个dummy thread来进行等待消息接收完成，真正的发送线程会立即退出
       if ((sending_thread->base.thread_state & _THREAD_DUMMY) != 0U) {
           //这里是在发送线程中执行，而挂起的dummy thread，因此不会卡在这里
           z_pend_thread(sending_thread, &mbox->tx_msg_queue, K_FOREVER);
           k_spin_unlock(&mbox->lock, key);
           return 0;
       }
   #endif
       //同步发送，如果没有匹配的线程接收，当前线程被加入tx_msg_queue中，等待超时。因为tx_msg是放到当前线程的交换数据内，这个操作就相当于将消息放入了邮箱
       int ret = z_pend_curr(&mbox->lock, key, &mbox->tx_msg_queue, timeout);

       return ret;
   }

接收
----

消息接收的流程和发送的流程是对称的，有下面两种情况： 1.
遍历邮箱\ ``tx_msg_queue``\ 中查看是否有消息，如果找到匹配的消息，接收线程接收处理该消息后通知发送线程恢复执行。
2.
如果没找到匹配的消息，将接收线程放入\ ``rx_msg_queue``\ ，挂起当前线程并等待接收消息超时。

代码详细分析如下：

::

   int k_mbox_get(struct k_mbox *mbox, struct k_mbox_msg *rx_msg, void *buffer,
              k_timeout_t timeout)
   {
       struct k_thread *sending_thread;
       struct k_mbox_msg *tx_msg;
       k_spinlock_key_t key;
       int result;

       //设置描述符中接收线程id为当前线程
       rx_msg->tx_target_thread = _current;

       //锁调度，开始处理消息接收
       key = k_spin_lock(&mbox->lock);

       //检查邮箱的tx_msg_queue中是否有可用消息
       _WAIT_Q_FOR_EACH(&mbox->tx_msg_queue, sending_thread) {
           //从线程交换数据中获取发送消息描述符
           tx_msg = (struct k_mbox_msg *)sending_thread->base.swap_data;

           //将收发消息描述进行匹配
           if (mbox_message_match(tx_msg, rx_msg) == 0) {
               //该消息和接收线程匹配，将发送消息的线程从邮箱tx_msg_queue中取出
               z_unpend_thread(sending_thread);

               k_spin_unlock(&mbox->lock, key);

               //处理数据，数据处理完后会让发送线程就绪并重新调度。如果这里buffer为空，会将数据保留，同时发送线程仍然处理挂起状态。
               result = mbox_message_data_check(rx_msg, buffer);

               return result;
           }
       }

       //没有匹配的消息，同时也不等待，退出本次接收
       if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
           k_spin_unlock(&mbox->lock, key);
           return -ENOMSG;
       }

       //如果没有匹配的消息，当前线程被加入rx_msg_queue中，等待超时
       _current->base.swap_data = rx_msg;
       result = z_pend_curr(&mbox->lock, key, &mbox->rx_msg_queue, timeout);

       //接收到消息进行消息处理，数据处理完后会让发送线程就绪并重新调度。如果这里buffer为空，会将数据保留，同时发送线程仍然处理挂起状态。
       if (result == 0) {
           result = mbox_message_data_check(rx_msg, buffer);
       }

       return result;
   }

消息匹配
--------

从前面的分析看到消息匹配是使用\ ``mbox_message_match``\ 对收发消息描述符进行比较，比较过程中主要有如下事项：

1. 比较收发的thread id是否匹配

2. 进行thread id交换

3. 进行info交换

4. 更新实际可以接收的size

5. 将消息的数据地址更新到接收消息描述符内

6. 将发送线程id更新到接收消息描述符中

::

   static int mbox_message_match(struct k_mbox_msg *tx_msg,
                      struct k_mbox_msg *rx_msg)
   {
       uint32_t temp_info;

       //匹配thread id
       if (((tx_msg->tx_target_thread == (k_tid_t)K_ANY) ||
            (tx_msg->tx_target_thread == rx_msg->tx_target_thread)) &&
           ((rx_msg->rx_source_thread == (k_tid_t)K_ANY) ||
            (rx_msg->rx_source_thread == tx_msg->rx_source_thread))) {

           //更新收发者的id，主要是給一方是K_ANY用，否则不会有变化
           rx_msg->rx_source_thread = tx_msg->rx_source_thread;
           tx_msg->tx_target_thread = rx_msg->tx_target_thread;

           //交换info
           temp_info = rx_msg->info;
           rx_msg->info = tx_msg->info;
           tx_msg->info = temp_info;

           //计算实际可以接收的数据
           if (rx_msg->size > tx_msg->size) {
               rx_msg->size = tx_msg->size;
           }

           //将消息的数据地址更新到接收消息描述符内
           rx_msg->tx_data = tx_msg->tx_data;
           rx_msg->tx_block = tx_msg->tx_block;
           if (rx_msg->tx_data != NULL) {
               rx_msg->tx_block.data = NULL;
           } else if (rx_msg->tx_block.data != NULL) {
               rx_msg->tx_data = rx_msg->tx_block.data;
           } else {
               /* no data */
           }

           //将发送线程id更新到接收消息描述符中
           rx_msg->_syncing_thread = tx_msg->_syncing_thread;

           return 0;
       }

       return -1;
   }

消息处理
--------

邮箱消息数据的处理有内部函数\ ``mbox_message_data_check``\ 和外部函数\ ``k_mbox_data_ge``\ t两个。在\ ``k_mbox_get``\ 传入的参数\ ``buffer=NULL``\ 时，\ ``mbox_message_data_check``\ 检查到时消息会被保留在邮箱。之后可以再通过\ ``k_mbox_data_get``\ 来读取该消息，如果此时\ ``buffer=NULL``\ 该消息就会从邮箱中取出丢弃。
代码分析如下：

::

   static int mbox_message_data_check(struct k_mbox_msg *rx_msg, void *buffer)
   {
       if (buffer != NULL) {
           //buffer不为NULL将数据读到buffer中
           k_mbox_data_get(rx_msg, buffer);
       } else if (rx_msg->size == 0U) {
           //buffer为空，且size为0，表示为空消息无需接收数据，直接将消息从邮箱中取出丢弃
           mbox_message_dispose(rx_msg);
       } else {
           //buffer为NULL且不是空消息，表示应用后续会用k_mbox_data_get取数据
       }

       return 0;
   }

   void k_mbox_data_get(struct k_mbox_msg *rx_msg, void *buffer)
   {
       //buffer为空，不需要消息数据，直接丢弃该消息
       if (buffer == NULL) {
           rx_msg->size = 0;
           mbox_message_dispose(rx_msg);
           return;
       }

       //不是空消息，将消息数据copy到buffer中
       if ((rx_msg->tx_data != NULL) && (rx_msg->size > 0U)) {
           (void)memcpy(buffer, rx_msg->tx_data, rx_msg->size);
       }

       //空消息，无需接收数据，直接将消息从邮箱中取出丢弃
       mbox_message_dispose(rx_msg);
   }

mbox_message_dispose再来看取出消息丢弃的流程

::

   static void mbox_message_dispose(struct k_mbox_msg *rx_msg)
   {
       struct k_thread *sending_thread;
       struct k_mbox_msg *tx_msg;

       // 消息被接收被通知的thread会变为NULL，因此不用再丢弃，这季节返回
       if (rx_msg->_syncing_thread == NULL) {
           return;
       }

       if (rx_msg->tx_block.data != NULL) {
           rx_msg->tx_block.data = NULL;
       }

       //通过将被通知thread设置为NULL，表示该消息已经被接收，并从邮箱丢弃
       sending_thread = rx_msg->_syncing_thread;
       rx_msg->_syncing_thread = NULL;


       //更新实际接收数据的大小
       tx_msg = (struct k_mbox_msg *)sending_thread->base.swap_data;
       tx_msg->size = rx_msg->size;

   #if (CONFIG_NUM_MBOX_ASYNC_MSGS > 0)
       //异步发送，消息接收完毕，邮箱中已删除该消息，使用信号量进行通知
       if ((sending_thread->base.thread_state & _THREAD_DUMMY) != 0U) {
           struct k_sem *async_sem = tx_msg->_async_sem;

           mbox_async_free((struct k_mbox_async *)sending_thread);
           if (async_sem != NULL) {
               k_sem_give(async_sem);
           }
           return;
       }
   #endif

       //同步发送，消息接收完毕，邮箱中已删除该消息，通知发送者退出等待状态，发送线程变为ready，并重新调度
       arch_thread_return_value_set(sending_thread, 0);
       z_mark_thread_as_not_pending(sending_thread);
       z_ready_thread(sending_thread);
       z_reschedule_unlocked();
   }

异步支持
--------

邮箱支持异步，在发送者发送消息后可以立即退出不用等到被接收后才退出。通过下面手段实现异步支持：
1. 初始化时创建指定数量的异步消息，里面包含一个发送消息描述符和一个用于等待消息被接收的dummy
thread

2. 发送异步消息时，将发送描述符存放在异步消息内，并将等待的同步_syncing_thread设置为dummy
thread，并指定通知接收完成的信号量

3. 接收消息完成后通过信号量通知接收已经完成

异步消息支持的数量受\ ``CONFIG_NUM_MBOX_ASYNC_MSGS``\ 配置限制。
这里我们主要分析异步消息初始化和发送的流程，其它都已经在前文分析的代码中进行了注释

异步消息
~~~~~~~~

异步消息的结构如下

::

   struct k_mbox_async {
       struct _thread_base thread;     //dummy thread
       struct k_mbox_msg tx_msg;   //发送消息描述符
   };


初始化
~~~~~~

初始化建立异步消息的stack，并将声明好的异步消息放入stack

::

   static int init_mbox_module(const struct device *dev)
   {
       ARG_UNUSED(dev);

       //声明CONFIG_NUM_MBOX_ASYNC_MSGS个异步消息
       static struct k_mbox_async __noinit async_msg[CONFIG_NUM_MBOX_ASYNC_MSGS];


       int i;

       for (i = 0; i < CONFIG_NUM_MBOX_ASYNC_MSGS; i++) {
           //将异步消息内的thread设置为dummy状态
           z_init_thread_base(&async_msg[i].thread, 0, _THREAD_DUMMY, 0);
           //将异步消息压入堆栈内
           k_stack_push(&async_msg_free, (stack_data_t)&async_msg[i]);
       }

       return 0;
   }

   //在Zephyr初始化的PRE_KERNEL_1阶段会调用init_mbox_module
   SYS_INIT(init_mbox_module, PRE_KERNEL_1, CONFIG_KERNEL_INIT_PRIORITY_OBJECTS);

异步发送
~~~~~~~~

异步发送和同步发送的主要流程一致，差别就是使用异步消息和dummy
thread进行等待，的代码如下：

::

   void k_mbox_async_put(struct k_mbox *mbox, struct k_mbox_msg *tx_msg,
                 struct k_sem *sem)
   {
       struct k_mbox_async *async;

       //分配异步消息
       mbox_async_alloc(&async);

       //初始化dummy thread的优先级
       async->thread.prio = _current->base.prio;

       //将发送消息存放在异步消息内
       async->tx_msg = *tx_msg;
       //将dummy_thread做为被通知thread进行等待，当前thread调用k_mbox_async_put可立即返回
       async->tx_msg._syncing_thread = (struct k_thread *)&async->thread;
       //将通知信号量保存在异步信息中
       async->tx_msg._async_sem = sem;

       //发送消息
       (void)mbox_message_put(mbox, &async->tx_msg, K_FOREVER);
   }

上面出现了对异步消息的堆栈的处理函数，其实就是对初始化时异步消息堆栈的处理，发送时pop出一个空的异步消息，接收完后将异步消息又加入到堆栈中

::

   static inline void mbox_async_alloc(struct k_mbox_async **async)
   {
       (void)k_stack_pop(&async_msg_free, (stack_data_t *)async, K_FOREVER);
   }

   static inline void mbox_async_free(struct k_mbox_async *async)
   {
       k_stack_push(&async_msg_free, (stack_data_t)async);
   }

从前面的分析我们知道初始化时空闲的异步消息只有\ ``CONFIG_NUM_MBOX_ASYNC_MSGS``\ 个。当空闲的异步消息用完后stack就为空了，此时再发送异步消息就会因为stack pop的特性发生阻塞，\ ``k_mbox_async_put``\ 被挂起，直到有异步消息被接收后push回stack。

参考
====

https://docs.zephyrproject.org/latest/reference/kernel/data_passing/mailboxes.html

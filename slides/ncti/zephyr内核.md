# zephyr内核

| 作者   | 修订 | 日期       |
| ------ | ---- | ---------- |
| 黄渊杰 |      | 2024-01-15 |



## 1、内存管理

k_malloc调用关系：`k_malloc->k_aligned_alloc->z_heap_aligned_alloc->k_heap_aligned_alloc`
k_free调用关系：`k_free->k_heap_free`

```c
K_HEAP_DEFINE(_system_heap, CONFIG_HEAP_MEM_POOL_SIZE);
#define _SYSTEM_HEAP (&_system_heap)
```

### [内存保护](https://lgl88911.github.io/2019/10/28/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E5%86%85%E5%AD%98%E4%BF%9D%E6%8A%A4/)

内存保护的过程就是在**进入用户模式线程前**，把将要执行的用户线程**thread stack**和domain内的**partition**内存区域通过MPU设置为可读写，其它内存区域MPU不允许访问。这样就达到了用户线程不能随意访问自己权限外内存区域的目的。

- 线程堆栈

  

- 内存域

- 堆栈保护

K_THREAD_STACK_DEFINE(thread_stack, STACKSIZE);





在zephyr下**用户线程**不能随意访问内存空间，用户线程之间除了通过内核交换数据外，还可以通过**内存域**的方法共享内存交换数据。

内存域的主要设计规则如下:

- partition: 内存的基本单元，规定了内存的起始地址和大小
- domain: partition的集合，一个domain可以包含多个partition，同一个partition可以出现在多个domain中
- 一个用户线程只能绑定一个domain，用户线程拥有绑定domain内所有partition的访问权限

```c
//定义4个partition
FOR_EACH(K_APPMEM_PARTITION_DEFINE, part0, part1, part2, part3, part4);
```

partition是如何放入section的, 这里分析两个宏K_APP_DMEM和K_APP_BMEM

```c
#define K_APP_DMEM(id) Z_GENERIC_SECTION(K_APP_DMEM_SECTION(id))
#define K_APP_BMEM(id) Z_GENERIC_SECTION(K_APP_BMEM_SECTION(id))

K_APPMEM_PARTITION_DEFINE
    
```

malloc的heap被放在z_malloc_partition中，这个partiton并没有授权给用户模式的线程，因此malloc无法在用户模式线程中使用。



### [Heap](https://lgl88911.github.io/2020/09/06/Zephyr%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8BHeap/)



### [Slab](https://lgl88911.github.io/2020/08/29/Zephyr%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8Bslab/)

Slab管理的内存块大小相同且固定。在一个系统中可以定义多个Slab，每个Slab能分配出来的内存块大小不一样





### 共享Heap

接口定义文件：zephyr/include/zephyr/sys/multi_heap.h
实现文件：zephyr/lib/os/multi_heap.c

```c

 //堆选择器回调，根据配置cfg,分配内存的大小size和对齐长度align从mheap中选出匹配的堆
typedef void *(*sys_multi_heap_fn_t)(struct sys_multi_heap *mheap, void *cfg,
				     size_t align, size_t size);

//初始化一个多堆heap，并注册堆选择器choice_fn
void sys_multi_heap_init(struct sys_multi_heap *heap,
			 sys_multi_heap_fn_t choice_fn);

//将堆heap和对应的用户数据user_data添加到多堆管理器mheap中
void sys_multi_heap_add_heap(struct sys_multi_heap *mheap, struct sys_heap *heap, void *user_data);

//按照配置cfg从多堆mheap中分配大小为bytes的内存
void *sys_multi_heap_alloc(struct sys_multi_heap *mheap, void *cfg, size_t bytes);

//按照配置cfg从多堆mheap中分配大小为bytes的内存，分配的内存需要按align长度对齐
void *sys_multi_heap_aligned_alloc(struct sys_multi_heap *mheap,
				   void *cfg, size_t align, size_t bytes);

//根据内存的地址addr，从多堆mheap中找出该内存属于哪个堆
const struct sys_multi_heap_rec *sys_multi_heap_get_heap(const struct sys_multi_heap *mheap,
							 void *addr);

//释放从多堆mheap中分配出的内存block
void sys_multi_heap_free(struct sys_multi_heap *mheap, void *block);
```



### Block



```c
zephyr/include/zephyr/sys/mem_blocks.h
zephyr/lib/os/mem_blocks.c
```

可以从指定的内存区域动态分配到内存块

- 同一个分配器中分配到的内存块大小都一样
- 可以同时分配和释放多个内存块
- 分配到一组的块可能不连续
- 管理区和内存区分离

和slab的区别， slab：

- 一次只能分配到一个slab

- 管理区在内存区中

  





### Demand Paging





## 2、线程管理



### 2.1优先级

zephyr用整数表示线程优先级，优先级可以为正也可以为负。整数值越小对应线程的优先级就越高，获得CPU的机会就越高。

优先级为负的是协作线程，协作线程一旦拿到CPU后会一直占用，k_sleep（让线程进入睡眠）或k_yield（引发一次调度）主动让出CPU。

![prio](https://lgl88911.github.io/2020/07/24/Zephyr%E7%BA%BF%E7%A8%8B%E4%BC%98%E5%85%88%E7%BA%A7%E7%AE%80%E4%BB%8B/prio.png)









### 2.2调度

#### 2.2.1 调度方式



#### 2.2.2 调度时机

- 就绪： 线程被放入read_q，线程运行时会保持在ready_q中。
- 等待内核对象：线程被放入内核对象的wait_q
- 睡眠：线程被放入timeout_list
- 挂起：线程脱离管理对象

![img](https://lgl88911.github.io/2021/09/27/Zephyr%E5%86%85%E6%A0%B8%E8%B0%83%E5%BA%A6%E4%B9%8B%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%901-%E7%BA%BF%E7%A8%8B%E6%B5%81%E8%BD%AC/sched_base.png)

- 就绪列队: 一个CPU只有一个read_q，用于存放管理已就绪的线程，这些线程等待使用CPU。

  就绪列队只有一个，其保存在_kernel.ready_q中，在z_sched_init中完成初始化

- 等待列队：一个内核对象有一个或者多个wait_q， 用于存放等待该内核对象的线程。

  等待列队会根据内核对象的数量有多个，在内核对象初始化的时候初始化等待列队

  

  

  Zephyr支持三种方法列队管理方法:

- 单链表
- 红黑树
- 多链表

API

```
_priq_run_add 		//将线程加入列队
_priq_run_remove	//将线程中列队中移除
_priq_run_best		//检查列队中哪个线程应该被选出占用资源

z_priq_wait_add 		//将线程加入列队
_priq_wait_remove	//将线程中列队中移除
_priq_wait_best		//检查列队中哪个线程应该被选出占用资源
```

#### 2.2.3 关键函数



```c
static ALWAYS_INLINE void queue_thread(void *pq, struct k_thread *thread)
static ALWAYS_INLINE void dequeue_thread(void *pq, struct k_thread *thread)
static inline bool z_is_thread_queued(struct k_thread *thread)
    
static void pend(struct k_thread *thread, _wait_q_t *wait_q, k_timeout_t timeout)
static inline void unpend_thread_no_timeout(struct k_thread *thread)
    
static void update_cache(int preempt_ok)
 
将最合适的线程选择出来并从wait_q移除，但不关心其是否在timeout_list中
struct k_thread *z_unpend1_no_timeout(_wait_q_t *wait_q)
将最合适的线程选择出来并从wait_q中移除，同时中止timeout检查
struct k_thread *z_unpend_first_thread(_wait_q_t *wait_q)
    
static ALWAYS_INLINE int z_swap(struct k_spinlock *lock, k_spinlock_key_t key)
static inline int z_swap_irqlock(unsigned int key)
static inline void z_swap_unlocked(void)
```



## 3、中断管理



IRQ_DIRECT_CONNECT(irq_p, priority_p, isr_p, flags_p)



## 4、[内核对象](https://lgl88911.github.io/2019/10/26/Zephyr%E7%94%A8%E6%88%B7%E6%A8%A1%E5%BC%8F-%E5%86%85%E6%A0%B8%E5%AF%B9%E8%B1%A1/)

zephyr的内核对象可以分为以下三种：

- 核心内核对象，例如信号量，线程，PIPE等
- 线程堆栈（由K_THREAD_STACK_DEFINE定义）
- subsystem中的设备驱动（struct device）实例
  详细的可以在gen_kobject_list.py中找到

```python
kobjects = OrderedDict([
    ("k_mem_slab", (None, False)),
    ("k_msgq", (None, False)),
    ("k_mutex", (None, False)),
    ("k_pipe", (None, False)),
    ("k_queue", (None, False)),
    ("k_poll_signal", (None, False)),
    ("k_sem", (None, False)),
    ("k_stack", (None, False)),
    ("k_thread", (None, False)),
    ("k_timer", (None, False)),
    ("_k_thread_stack_element", (None, False)),
    ("device", (None, False)),
    ("sys_mutex", (None, True)),
    ("k_futex", (None, True))
])
```

```c
struct z_object {
	void *name;
	uint8_t perms[CONFIG_MAX_THREAD_BYTES];
	uint8_t type;
	uint8_t flags;
	union z_object_data data;
} __packed __aligned(4);
```



静态内核对象struct _k_object是隐式生成，当定义的内核对象变量被编译入elf后，gen_kobject_list.py脚本会扫描elf文件，生成kobject_hash.gperf文件，然后由gperf工具生成kobject_hash.c，目标是建立一个内核对象hash查询函数。对于某个内核对象来说，这个过程是在elf中查找内核对象，并将它们的内存地址放在内核对象元hash表中，达到生成struct _k_object的目的。

**限制条件**

- 内核对象必须位于内核保留内存中(不会被用户线程通过内存地址直接访问)
- 内核对象必须是全局变量（才能出现在elf符号表中，而被脚本扫描到）







```
irq_lock() ISR中调用会锁住所有中断，线程中调用只会在当前线程下锁住所有中断，返回一个key值，unlock时需要传递该key值
irq_unlock(key) 解锁指定key值，lock可已多次执行会被计数引用，unlock需要执行对应的次数才能真正的解锁
irq_enable(irq) 使能指定irq
irq_disable(irq) 禁止指定的irq

irq_is_enabled(irq) 检查指定的irq是否被使能
bool k_is_in_isr(void) 检查当前代码是否执行在isr中，如果是返回true，如果执行在线程中返回false
int k_is_preempt_thread(void) 检查当前代码是否执行在可抢占式线程中，如果是返回非0，如果执行在ISR或者协程中返回0
static bool k_is_pre_kernel(void) 检查当前代码是否执行在启动阶段，如果是post-kernel阶段之前返回true, 如果是post-kernel期间或者之后返回false
```

## 5、定时器

kernel/timeout.c

![timeout](https://lgl88911.github.io/2020/04/26/Zephyr%E5%86%85%E6%A0%B8Timeout%E6%A8%A1%E5%9D%97%E7%AE%80%E4%BB%8B/timeout.png)

Timeout本身维护一个双向链表，链表的timeout_list, 链表内等待timeout的节点已等待的**tick数从小到大排序**，某一个节点要等待的dick数是**从链表的头一直累加到该节点**:

```c

```

## 6、[ISR](https://lgl88911.github.io/2020/03/02/Zephyr%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F-%E5%AE%9E%E7%8E%B0/)





### **生成中断向量表**

![zephyrvector](https://lgl88911.github.io/2020/03/02/Zephyr%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F-%E5%AE%9E%E7%8E%B0/zephyrvector.png)

Zephyr采用了两阶段链接，第一阶段链接生成zephyr_prebuild.elf，该elf中中断向量由zephyr/arch/common/isr_tables.c生成，只是占位。在第一阶段ISR_CONNECT和ISR_DERICT_CONNECT注册的中断向量都放在zephyr_prebuild.elf的.intList段中，第二阶段链接时，会通过objcopy从zephyr_prebuild.elf中dump出.intList,然后由gen_isr_table.py解析生成新的isr_table.c，生成过程就是将direct isr放入新的硬件中断向量表_irq_vector_table，普通的isr放入新的软件中断向量表，生成新的**isr_table.c**，然后新的isr_table.c重新和zephyr的lib链接生成zephyr.elf，在该过程中就会替换掉之前的中断向量表。



### ISR执行过程

### zephyr中断向量表

![zephyrint](https://lgl88911.github.io/2020/03/02/Zephyr%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F-%E5%AE%9E%E7%8E%B0/zephyrint.png)



Zephyr维护硬、软两张中断向量表，_**irq_vector_table**为硬中断向量表，也就是基础概念中提到的中断向量表，_irq_vector_table的起始地址和exception 16(IRQ0)对齐。通过IRQ_DIRECT_CONNECT安装的ISR会直接保存在该向量表中，当中断发生时直接执行中断向量表中保存的ISR函数。通过IRQ_CONNECT/irq_connect_dynamic安装的ISR会保存在软件中断向量表_**sw_isr_tabl**e中，当中断发生时，会执行硬中断向量表中默认的_isr_wapper在软中断向量表_sw_isr_table中进行查表，查到对应的ISR函数并执行。如果某个IRQ没有安装任何ISR，硬中断向量表中默认放置_isr_wapper，软中断表中默认z_irq_spurious，当中断发生时会查表到z_irq_spurious执行，在z_irq_spurious进入fault异常。

### 直接ISR

访问中断入口函数test_direct_isr，直接执行

### 普通ISR

以前面的中断向量表示例为例当IRQ3发生中断时, 访问中断入口函数ISR_WRAPPER，也就是_isr_wrapper

### API

```c
IRQ_CONNECT(irq_p, priority_p, isr_p, isr_param_p, flags_p)
static inline int irq_connect_dynamic(unsigned int irq, unsigned int priority,void (routine)(void parameter), void *parameter, u32_t flags)
 /*
IRQ发生时ISR会被直接调用，是实际的将ISR函数地址放入到硬件指向的中断向量表，因此和普通的ISR比会有下面的不同
    不支持ISR参数
    只能使用中断堆栈
    ISR退出时可选是否切换上下文
    ISR退出时可选是否退出IDLE省电状态
    中断锁状态不改变
 */
IRQ_DIRECT_CONNECT(irq_p, priority_p, isr_p, flags_p)
控制
irq_lock() ISR中调用会锁住所有中断，线程中调用只会在当前线程下锁住所有中断，返回一个key值，unlock时需要传递该key值
irq_unlock(key) 解锁指定key值，lock可已多次执行会被计数引用，unlock需要执行对应的次数才能真正的解锁
irq_enable(irq) 使能指定irq
irq_disable(irq) 禁止指定的irq

状态
irq_is_enabled(irq) 检查指定的irq是否被使能
bool k_is_in_isr(void) 检查当前代码是否执行在isr中，如果是返回true，如果执行在线程中返回false
int k_is_preempt_thread(void) 检查当前代码是否执行在可抢占式线程中，如果是返回非0，如果执行在ISR或者协程中返回0
static bool k_is_pre_kernel(void) 检查当前代码是否执行在启动阶段，如果是post-kernel阶段之前返回true, 如果是post-kernel期间或者之后返回false


```





## 7、[工作队列](https://lgl88911.github.io/2022/08/20/Zephyr%E5%86%85%E6%A0%B8%E5%AF%B9%E8%B1%A1-%E5%B7%A5%E4%BD%9C%E5%88%97%E9%98%9F/)

工作列队由一个专用线程和列队组成，专用线程以先进先出的方式处理发送到列队中的工作项。

只要内存足够在Zephyr可以创建任意数量的工作列队，一个工作列队就会有一个工作线程，工作线程的优先级可以配置为协助或者抢占。当列队中无工作项时工作线程处于睡眠状态。当使用者向工作列队发送工作项时，工作项被加入到列队同时通知工作线程处理，工作线程从列队中取出工作项执行，工作线程在执行工作项之间会使用`k_yield`以防止协作类型的工作线程一直占用CPU饿死其它线程。

一个工作列队的三大要素：

- 工作项：中断服务程序或高优先级线程的**非紧急事务**
- 列队：用于保存还没处理的工作项
- 工作线程：处理工作项中携带的事务

工作项在生命周期内有如下状态：

- 悬空：被发送者初始化
- 排队(**queued**)：`K_WORK_QUEUED` 工作项被发送放到列队中，但还未执行
- 预约(**scheduled**): `K_WORK_DELAYED` 延迟工作项(后文说明)在等待超时
- 运行(**running**)：`K_WORK_RUNNING` 工作项正在被工作线程处理
- 弃用(**canceling**): `K_WORK_CANCELING` 弃用正在被处理的工作项

工作项可能会同时拥有以上几种状态的组合，例如启用一个正在执行的工作项，这个工作项的状态就是`K_WORK_RUNNING | K_WORK_CANCELING`



#### 用户定义工作列队

只要内存足够用户可以定义任意数量的工作列队，创建一个工作列队需要通过初始化和启动两步完成，工作列队的线程是在启动工作列队时才创建

```c
/* 工作列队线程的属性 */
#define USER_WQ_STACK_SIZE 4096
#define USER_WQ_PRIORITY 5

/* 定义工作列队线程的堆栈 */
K_THREAD_STACK_DEFINE(user_wq_stack_area, USER_WQ_STACK_SIZE);

/* 初始化工作列队 */
struct k_work_q user_work_q;
k_work_queue_init(&user_work_q);

/* 启动工作列队 */
k_work_queue_start(&user_work_q, user_wq_stack_area,
                    K_THREAD_STACK_SIZEOF(user_wq_stack_area), USER_WQ_PRIORITY,
                    NULL);
```



```c
/* 初始化工作项，为其指定工作函数user_work_handler */
struct k_work work;
k_work_init(&work, user_work_handler);

/* 将工作项发送到 user_work_q的工作列队处理 */
k_work_submit_to_queue(&user_work_q, &work);
```

#### 系统工作列队

zephyr在`POST_KERNEL`初始化阶段会创建一个系统工作列队`k_sys_work_q`

```c
SYS_INIT(k_sys_work_q_init, POST_KERNEL, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT);
```

该工作列队的线程优先级通过`CONFIG_SYSTEM_WORKQUEUE_PRIORITY`配置，默认`-1`，属于不可抢占的协作线程。通过`CONFIG_SYSTEM_WORKQUEUE_NO_YIELD`配置系统工作列队在处理完一个工作项后是否执行`k_yield`，**默认为`n`表示需要执行**

#### 对工作项的操作

无论是提交到系统工作列队还是用户工作列队的工作项都可以使用下列API进行操作

```c
struct k_work_sync sync;

/* 等待work执行完成后才返回 */
k_work_flush(&work, &sync);

/* 异步取消work立即返回，列队等待的work将被移除，已经再执行的work依然执行完 */
k_work_cancel(&work);

/* 同步取消work，列队等待的work将被移除，已经再执行的work依然执行完,等到执行完后才会返回 */
k_work_cancel(&work, &sync);
```



### 延迟工作项

延迟工作项会在指定时间后才提交到工作列队中。

```c
/* 初始化一个延迟工作项 */
struct k_work_delayable delay_work;k_work_init_delayable(&delay_work, delay_work_handler);
/* 300ms后将延迟工作项提交到user_work_q工作列队中*/
k_work_schedule_for_queue(&user_work_q, &delay_work, K_MSEC(300));
//对一个处于预约状态尚未提交的延迟工作项再次做`k_work_schedule/k_work_schedule_for_queue`不会改变其预约时间

/* 将系统工作列队delay_work的预约时间修改为3000ms */
k_work_reschedule(&delay_work, K_MSEC(3000));
/* 将用户工作列队user_work_q中delay_work的预约时间修改为3000ms */
k_work_reschedule_for_queue(&user_work_q, &delay_work, K_MSEC(3000));
```

### 取消延迟工作项

取消分为三种状态处理：

- 工作项处在预约等待状态，取消预约定时器
- 工作项处于列队等待状态，从列队中移除
- 工作项处于运行状态，标记取消，不会中止运行

取消延迟工作项分为异步和同步两种，异步取消只是通知取消不会等待真正的取消就会退出。例如如果取消一个正在运行的工作项，同步取消函数会等到运行完毕后才会返回。

**异步取消**

```c
k_work_cancel_delayable(&delay_work);
```

**同步取消**

```c
k_work_cancel_delayable_sync(&delay_work);
```

#### 等待执行

执行等待执行时如果延迟工作项在预约状态将取消预约并立即提交到工作列队，并等到执行完成后才返回

```c
struct k_work_sync sync;

/* 延迟工作项被立即提交到工作列队，等待执行完后才返回 */
k_work_flush_delayable(&delay_work, &sync);
```



## 8、subsys

### [logging](https://lgl88911.gitee.io/2019/03/10/Zephyr-log%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86/)



![logsys](https://lgl88911.gitee.io/2019/03/10/Zephyr-log%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86/logsys.png)



#### active

```c
static void msg_process(struct log_msg *msg, bool bypass)

    
```



#### [filter](https://lgl88911.github.io/2019/04/07/Zephyr-log%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86%E4%B9%8Bfilter/)

![filter](https://lgl88911.github.io/2019/04/07/Zephyr-log%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86%E4%B9%8Bfilter/filter.png)



























参考文献：

[1] [Zephyr内核调度之调度方式与时机](https://lgl88911.github.io/2020/08/23/Zephyr内核调度之调度方式与时机/)
[2] [Zephyr内核调度之列队管理算法](https://lgl88911.github.io/2020/08/09/Zephyr内核调度之列队管理算法/)
[3] [Zephyr线程生命周期及状态机](https://lgl88911.github.io/2020/07/31/Zephyr线程生命周期及状态机/)
[4] [Zephyr线程阻塞和超时机制分析](https://lgl88911.github.io/2020/05/17/Zephyr线程阻塞和超时机制分析/)
[5] [Zephyr线程优先级简介](https://lgl88911.github.io/2020/07/24/Zephyr线程优先级简介/)

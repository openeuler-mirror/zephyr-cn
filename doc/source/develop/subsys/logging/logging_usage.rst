.. _logging_logging_usage:

Zephyr日志系统使用指南
#########################

Zephyr包含功能强大的日志系统，以通用的接口的方式提供给开发人员使用。日志消息通过前端传递，然后由后端进行处理。
Zephyr的日志系统具有以下特性：

* 延迟异步日志，将处理日志转移到线程中处理，减少发送消息的耗时。

* 最多支持9个后端
  
  * 可自定义后端
  
  * 同时支持多个后端：串口，文件系统，网络，调试工具等
  
  * 可在允许时激活或者关闭终端

* 支持日志过滤
  
  * 支持运行时指定模块或实例的日志等级
  
  * 支持编译时指定模块的日志等级

* 支持日志等级
  
  * 支持debug，info，waring，error 4个严重等级的日志

* 附加消息标签
  
  * 支持模块标签
  
  * 支持实例标签
  
  * 支持附加时间戳，最大可支持64位时间戳
  
  * 支持全局标签

* 支持完整的C99格式化处理

* 支持Raw Data显示功能

* 支持”瞬态“字符串

* 支持Panic mode输出

* printk可以被重定位到日志系统

模块日志
========

模块日志可以将一个或几个文件当作一个模块，在这些文件中的日志消息输出时都有相同的模块标签和相同的日志过滤等级。

基本的使用方法
~~~~~~~~~~~~~~~~

首先在``prj.conf``中开启日志系统

.. code:: 

    CONFIG_LOG=y

代码中通过下面两步就可以
1. 在模块文件中使用 ``LOG_MODULE_REGISTER`` 进行注册  
2. 使用 ``LOG_*`` 或 ``LOG_HEXDUMP_*`` 进行日志记录
  
示例代码如下

.. code:: c

    #include <logging/log.h>
    // 注册simple模块
    LOG_MODULE_REGISTER(simple, 4);

    void simple_logging(void)
    {

        uint8_t data[] = { 1, 2, 3, 4, 5, 6, 7, 8,
            9, 10, 11, 12, 13, 14, 15, 16,
            17, 18, 19, 20, 21, 22, 23, 24,
            25, 26, 27, 28, 29, 30, 31, 32,
            33, 34, };

        // 以不同等级输出日志
        LOG_ERR("Error message example.");
        LOG_WRN("Warning message example.");
        LOG_INF("Info message example.");
        LOG_DBG("Debug message example.");

        //Dump二进制数据
        LOG_HEXDUMP_ERR(data, sizeof(data),"Error hexdump example:");
    }


其输出结果为

.. code::

    [00:06:30.554,273] <err> simple: Error message example.
    [00:06:30.554,300] <wrn> simple: Warning message example.
    [00:06:30.554,309] <inf> simple: Info message example.
    [00:06:30.554,322] <dbg> simple: simple_logging: Debug message example.
    [00:00:25.371,571] <err> simple: Error hexdump example:
    01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 |........ ........
    11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f 20 |........ .......
    21 22 |!"


如果同时要在其它文件也使用simple作为模块标签，按照下面方法使用 ``LOG_MODULE_DECLARE`` 声明模块后即可使用

.. code:: c

    #include <logging/log.h>
    LOG_MODULE_DECLARE(simple);

    void simple_co_logging(void)
    {
        LOG_ERR("Error message example co.");
        LOG_WRN("Warning message example co.");
        LOG_INF("Info message example co.");
        LOG_DBG("Debug message example co.");
    }


接口简要说明
~~~~~~~~~~~~

\ ``LOG_MODULE_REGISTER`` 是可变参宏，第一个参数是模块名标签，上例为simple。第二个参数表示该模块的build-in日志等级，该参数是可选的，上例是4代表build-in日志等级为DEBUG。如果不指定 ``LOG_MODULE_REGISTER(simple)`` 就是默认为3代表为INFO等级。
当有多个文件属于同一个模块时，只能在一个文件中使用 ``LOG_MODULE_REGISTER`` 注册模块，其它文件使用 ``LOG_MODULE_DECLARE`` 进行模块声明，这样这些文件就属于同一个模块。
开发者可以使用以下接口进行日志输出。

模块日志输出，格式化输出字符串到日志系统，参数格式和 ``printf`` 一致，格式化字符串符合C99规定的标准。模块日志输出的宏会自动在字符串末尾加入 ``\r\n`` 

.. code:: c

    LOG_ERR(...)
    LOG_WRN(...)
    LOG_INF(...)
    LOG_DBG(...)


模块日志输出，原始数据16进制输出到日志系统

.. code:: c

    LOG_HEXDUMP_ERR(_data, _length, _str)
    LOG_HEXDUMP_WRN(_data, _length, _str)
    LOG_HEXDUMP_INF(_data, _length, _str)
    LOG_HEXDUMP_DBG(_data, _length, _str)


\ ``_data`` 为要输出的原始数据， ``_length`` 为原始数据的长度， ``_str`` 为提示字符串。会将 ``_data`` 内的数据以16进制的形式输出，并显示其对应的ascii码。
以上四种类型的宏分别对应4个严重等级的消息：

* ``*_ERR`` ：ERROR消息，指示严重错误，例如无法恢复的错误。输出消息中会带有"err"字符串标签。  
  
* ``*_WRN`` ：WARNING消息，指示不严重的异常情况。输出消息中会带有"wrn"字符串标签。  
  
* ``*_INF`` ：INFO消息，指示一般的信息。输出消息中会带有"inf"字符串标签。  
  
* ``*_DBG`` ：DEBUG消息，调试信息。输出消息中会带有"dbg"字符串标签。

实例日志
========

当一个模块会以多实例运行时，不同的实例显示出来的模块标签都是同一个，这将导致无法识别到底时那个实例的日志输出。为解决该问题Zephyr日志系统提供了多实例支持，可以在实例级别进行过滤。

使用方法
~~~~~~~~

按照如下步骤使用实例日志：

1. 在模块文件中使用 ``LOG_MODULE_REGISTER`` 进行注册
  
2. 使用 ``LOG_INSTANCE_PTR_DECLARE`` 在实例结构中指定日志的结构
  
3. 使用 ``LOG_INSTANCE_REGISTER`注册实例，并使用`LOG_INSTANCE_PTR_INIT`` 进行日志实例初始化。
  
4. 使用 ``LOG_INST_*`` 或 ``LOG_INST_HEXDUMP_*`` 进行日志记录。
  

示例代码 ``sample_instance.h`` 如下

.. code:: c

    #include <logging/log.h>

    /* 1. 在实例结构中用LOG_INSTANCE_PTR_DECLARE指定日志的结构 */
    struct sample_instance {
        LOG_INSTANCE_PTR_DECLARE(log);
        uint32_t cnt;
    };


示例代码 ``sample_instance.c`` 如下

.. code:: c

    #include "sample_instance.h"

    /* 2.注册模块日志，模块名为module_inst，等级为DEBUG */
    #include <logging/log.h>
    #define SAMPLE_INSTANCE_MODULE_NAME module_inst
    LOG_MODULE_REGISTER(SAMPLE_INSTANCE_MODULE_NAME, 4);


    /* 该宏用于获取实例结构体遍历指针 */
    #define SAMPLE_INSTANCE_PTR(idx) &sample_##idx##_info

    /* 该宏用于生成实例，为每个实例注册日志，并进行初始化 */
    #define SAMPLE_INSTANCE_DEVICE(idx) \
        LOG_INSTANCE_REGISTER(SAMPLE_INSTANCE_MODULE_NAME, inst##idx, 3); \
        struct sample_instance sample_##idx##_info = {           \
            LOG_INSTANCE_PTR_INIT(log, SAMPLE_INSTANCE_MODULE_NAME, inst##idx)       \
        };

    /* 最多支持3个实例 */
    #define SAMPLE_INSTANCE_NUM 3

    /* 3.使用宏注册和初始化实例日志 */
    SAMPLE_INSTANCE_DEVICE(0)
    SAMPLE_INSTANCE_DEVICE(1)
    SAMPLE_INSTANCE_DEVICE(2)


    static struct sample_instance *sample_instace_ptr[]={
        SAMPLE_INSTANCE_PTR(0), 
        SAMPLE_INSTANCE_PTR(1), 
        SAMPLE_INSTANCE_PTR(2)};

    struct sample_instance *sample_instance_open(uint8_t id)
    {
        if(id < SAMPLE_INSTANCE_NUM){
            sample_instace_ptr[id]->cnt = 10;
            return sample_instace_ptr[id];
        }

        return NULL;
    }

    int sample_instance_close(struct sample_instance *inst)
    {
        if(inst == NULL){
            return -1;
        }
        return 0;
    }

    int sample_instance_do(struct sample_instance *inst)
    {
        if(inst == NULL){
            return -1;
        }

        /* 4.进行实例日志输入 */
        LOG_INST_INF(inst->log, "inst %p counter_value: %d", inst, inst->cnt);
        inst->cnt++;
        return 0;
    }


当头文件中的函数需要加入到模块实例中时，需要在头文件的函数体内使用 ``LOG_LEVEL_SET`` 指定日志等级，再调用 ``LOG_INST_*`` 进行日志输出。示例代码添加再 ``sample_instance.h`` 如下:

.. code:: c

    static inline void sample_instance_header_do(struct sample_instance *inst)
    {
        LOG_LEVEL_SET(LOG_LEVEL_INF);
        LOG_INST_INF(inst->log, "inst %p header do %d.", inst, inst->cnt);
        inst->cnt++;
    }


该示例演示了一个可多实例工作的sample_instance模块，通过 ``sample_instance_open`` 开启一个新的实例， ``sample_instance_do`` 来执行指定实例时通过日志系统输出自己的内部计数， ``sample_instance_header_do`` 指定了一个INFO等级的实例日志输出。该模块在日志系统中的模块名为module_inst，为了区分不同的实例的打印使用了实例日志，测试代码如下：

.. code:: c

    void instance_logging(void)
    {
        struct sample_instance *inst0 = sample_instance_open(0);
        struct sample_instance *inst1 = sample_instance_open(1);
        struct sample_instance *inst2 = sample_instance_open(2);

        sample_instance_do(inst0);
        sample_instance_do(inst1);
        sample_instance_do(inst2);
        sample_instance_header_do(inst0);

        sample_instance_close(inst0);
        sample_instance_close(inst1);
        sample_instance_close(inst2);
    }


其输出结果为：

.. code:: 

    [00:00:25.731,861] <inf> module_inst.inst0: inst 0x3fc8c364 counter_value: 10
    [00:00:25.731,901] <inf> module_inst.inst1: inst 0x3fc8c36c counter_value: 10
    [00:00:25.731,904] <inf> module_inst.inst2: inst 0x3fc8c374 counter_value: 10
    [00:00:25.731,922] <inf> module_inst.inst0: inst 0x3fc8c364 header do 11.

从结果能够看到在模块标签module_inst后又多了一个实例标签inst\*，这样即使是同一份代码，也能够根据实例标签判断是哪个实例产生的日志。

**注意**：实例日志的注册宏 ``LOG_INSTANCE_REGISTER`` 要求参数有实例名，该宏是在编译期间就展开了，因此实例名必须在编译期就确定，无法运行时生成。这就要求使用实例日志的多实例模块必须要预先知道最大实例数，并指定好日志系统的实例名，而无法在运行时根据实际的实例数量动态生成。

接口说明
~~~~~~~~

实例日志的格式化输出接口和原始数据16进制输出接口如下

.. code:: c

    LOG_INST_ERR(_log_inst, ...)
    LOG_INST_WRN(_log_inst, ...)
    LOG_INST_INF(_log_inst, ...)
    LOG_INST_DBG(_log_inst, ...)
    LOG_INST_HEXDUMP_ERR(_log_inst, _data, _length, _str)
    LOG_INST_HEXDUMP_WRN(_log_inst, _data, _length, _str)
    LOG_INST_HEXDUMP_INF(_log_inst, _data, _length, _str)
    LOG_INST_HEXDUMP_DBG(_log_inst, _data, _length, _str)


只比模块日志多指定一个 ``_log_inst`` 的实例参数，其它使用方法一样

日志过滤
========

Zephyr中存在大量的模块日志，在系统编译或运行时均可进行日志过滤，编译时的过滤是指定日志最小的等级，一旦设定运行时无法改变。运行时过滤的等级只能大于或者等于编译时指定的过滤等级。例如编译时指定了过滤等级为WARNING，那么运行时只允许设置`*_WRN`和`*_ERR`输出的消息，无法通过运行时设置让`*_INF`和`*_DBG`的日志显示出来。对应的关系如下。

编译时过滤等级
~~~~~~~~~~~~~~

+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| 编译时过滤等级   | ``LOG_ERR/LOG_HEXDUMP_ERR`` | ``LOG_WRN/LOG_HEXDUMP_WRN`` | ``LOG_INF/LOG_HEXDUMP_INF`` | ``LOG_DBG/LOG_HEXDUMP_DBG`` |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| LOG_LEVEL_NONE 0 | OFF                         | OFF                         | OFF                         | OFF                         |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| LOG_LEVEL_ERR 1U | ON                          | OFF                         | OFF                         | OFF                         |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| LOG_LEVEL_WRN 2U | ON                          | ON                          | OFF                         | OFF                         |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| LOG_LEVEL_INF 3U | ON                          | ON                          | ON                          | OFF                         |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+
| LOG_LEVEL_DBG 4U | ON                          | ON                          | ON                          | ON                          |
+------------------+-----------------------------+-----------------------------+-----------------------------+-----------------------------+

现有模块的编译时过滤等级
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Zephyr的代码内有大量的日志输出，默认情况下这些日志都被限制在INFO等级，其DEBUG等级的日志不会输出到后端，在开发调试过程中遇到问题时希望查看模块DEBUG等级的日志进行诊断。另外就是在产品发布时会通过提高日志过滤等级只让ERROR等级输出甚至不输出。为达到这些目标就需要通过配置修改模块日志的默认过滤等级来开启日志。这里以I2C的为例说明如何指定模块的默认过滤等级：

1. 找到模块代码，I2C的在 ``drivers/i2c/`` 目录下
  
2. 打开任意有使用 ``LOG_MODULE_REGISTER`` 的源文件，例如 ``i2c_esp32.c`` 查看其配置项为 ``CONFIG_I2C_LOG_LEVEL`` 
  
  .. code:: c

      LOG_MODULE_REGISTER(i2c_esp32, CONFIG_I2C_LOG_LEVEL);

  
3. 使用`配置项_等级`模式在 ``prj.conf`` 中进行配置，可用配置项如下
  
.. code:: 

  # 不显示任何LOG
  CONFIG_I2C_LOG_LEVEL_OFF=y
  # 显示ERROR等级LOG
  CONFIG_I2C_LOG_LEVEL_ERR=y
  # 显示WARNING及以上等级LOG
  CONFIG_I2C_LOG_LEVEL_WRN=y
  # 显示INFO及以上等级LOG
  CONFIG_I2C_LOG_LEVEL_INF=y
  # 显示DEBUG及以上等级LOG
  CONFIG_I2C_LOG_LEVEL_DBG=y    

  
**注意：只能5选1**，例如想要显示所有的I2C日志，就在 ``prj.conf`` 中添加
  
.. code:: 

    CONFIG_I2C_LOG_LEVEL_DBG=y 
  
对于其它的模块也是一样的模式，例如想要关闭FLASH中日志，就在 ``prj.conf`` 中添加
  
.. code:: 

    CONFIG_FLASH_LOG_LEVEL_OFF=y 


自定义模块的编译时过滤等级
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

前面演示过编译时指定的过滤等级由 ``LOG_MODULE_REGISTER`` 的最后一个参数指定，例如

.. code:: c

    LOG_MODULE_REGISTER(module_name, 3)


\ ``LOG_MODULE_REGISTER`` 是变参宏，当没有显式的指定过滤等级时，使用 ``CONFIG_LOG_DEFAULT_LEVEL`` 配置的过滤等级，默认情况下该配置项为3，也就是INFO等级。
例如

.. code:: c

    LOG_MODULE_REGISTER(module_name)

**改变默认等级**
在 ``prj.conf`` 中添加下面配置5选一对默认过滤等级进行配置：

.. code:: 

  # 不显示任何LOG
  CONFIG_LOG_DEFAULT_LEVEL_OFF=y
  # 显示ERROR等级LOG
  CONFIG_LOG_DEFAULT_LEVEL_ERR=y
  # 显示WARNING及以上等级LOG
  CONFIG_LOG_DEFAULT_LEVEL_WRN=y
  # 显示INFO及以上等级LOG
  CONFIG_LOG_DEFAULT_LEVEL_INF=y
  # 显示DEBUG及以上等级LOG
  CONFIG_LOG_DEFAULT_LEVEL_DBG=y 


**自定义模块过滤等级**
前面的内容可以看到，在自定义的模块中我们是使用的数字直接指定编译时的过滤等级，当我们需要修改时，需要到源文件内进行修改，这样做并不方便。我们可以通过导入日志Kconfig模板的方式让自定义的模块也能通过配置项进行编译时默认过滤等级配置。方法如下：
- 为自定义模块添加Kconfig文件，示例代码 ``Kconfig.logging`` 内容如下
  
.. code:: 

  # 指定模块
  module = LOGGING_SAMPLE
  
  # 指定在menuconfig中配置项显示的字符串
  module-str = logging_sample
  
  # 导入日志配置模板
  source "subsys/logging/Kconfig.template.log_config"

  
- 在自定义模块内应用配置项，示例代码内容如下
  
  .. code:: c

    #include <logging/log.h>
    LOG_MODULE_REGISTER(logging_sample, CONFIG_LOGGING_SAMPLE_LOG_LEVEL);

  
- 在 ``prj.conf`` 中进行配置
  
  .. code:: 

    CONFIG_LOGGING_SAMPLE_LOG_LEVEL_ERR=y
  
  
  如果不进行配置，将默认为INFO等级。
  
可选配置项为

.. code:: 

  # 不显示任何LOG
  CONFIG_LOGGING_SAMPLE_LOG_LEVEL_OFF=y
  # 显示ERROR等级LOG
  CONFIG_LOGGING_SAMPLE_LOG_LEVEL_ERR=y
  # 显示WARNING及以上等级LOG
  CONFIG_LOGGING_SAMPLE_LOG_LEVEL_WRN=y
  # 显示INFO及以上等级LOG
  CONFIG_LOGGING_SAMPLE_LOG_LEVEL_INF=y
  # 显示DEBUG及以上等级LOG
  CONFIG_LOGGING_SAMPLE_LOG_LEVEL_DBG=y 


运行时日志过滤
~~~~~~~~~~~~~~

在Zephyr通过编译时过滤可以限制模块最小的过滤等级，但在最小等级之上的模块日志还是会输出。我们在调试时希望控制相关模块日志进行输出分析，不希望其它无关的日志都输出，这种情况下需要在运行时可以设置模块的日志过滤等级。需要注意的是运行时过滤的等级只能大于或者等于编译时指定的过滤等级。

代码控制过滤等级
^^^^^^^^^^^^^^^^

运行时过滤提供了过滤日志更多的灵活性，让我们在代码里面可以根据触发条件对日志过滤等级进行修改，更方便调试。

控制运行时日志使用的接口都在头文件 ``include/logging/log_ctrl.h`` 中声明，设置过滤等级的函数为 ``log_filter_set`` 。示例代码中 ``logging_filter_test`` 函数演示了如何使用 ``log_filter_set`` 进行过滤：

.. code:: c

    /* 获取source id的函数需要自己实现 */
    static int16_t log_source_id_get(const char *name)
    {
        /* 使用log_source_name_get取出每个source的名称做对比，确认source的id */
        for (int16_t i = 0; i < log_src_cnt_get(CONFIG_LOG_DOMAIN_ID); i++) {
            if (strcmp(log_source_name_get(CONFIG_LOG_DOMAIN_ID, i), name)
                == 0) {
                return i;
            }
        }

        return -1;
    }

    static int logging_filter_test(const struct shell *shell, size_t argc, char **argv)
    {
        uint32_t level = strtol(argv[1], NULL, 0);
        uint32_t act_level = 0;

        shell_print(shell, "Set filter level %d", level);
        act_level = log_filter_set(NULL,     /* 第一个参数为backend，如果为NULL表示对所有的backend都进行设置 */
                        0,                    /* 第二个参数为domain_id，目前不支持多dimain，直接给0 */
                        log_source_id_get(STRINGIFY(logging_sample)), /* 第三个参数为source_id，代表的是要设置那个模块的等级，这里我们要设置logging_sample */
                        level);                /* 第四个参数为level，表示要设置的过滤等级 */

        /* 返回的act_level为实际生效的level */
        shell_print(shell, "Set filter level %d act is %d", level, act_level);
        return 0;
    }


\ ``log_filter_set`` 的 ``level`` 参数可选如下，返回值为实际实际生效

.. code:: c

    #define LOG_LEVEL_NONE 0U
    #define LOG_LEVEL_ERR  1U
    #define LOG_LEVEL_WRN  2U
    #define LOG_LEVEL_INF  3U
    #define LOG_LEVEL_DBG  4U


\ ``log_filter_set`` 的返回值为实际生效的等级，当 ``level`` 设置的等级低于编译时等级时，将按照编译时默认等级进行设置生效。

通过Shell命令控制日志系统
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当启用zephyr shell命令时，可以通过shell的log命令进行日志过滤等级设置，后端设置，状态查看等。log支持的命令如下

.. code::

    uart:~$ log help
    log - Commands for controlling logger
    Subcommands:
    backend :Logger backends commands.
    disable :'log disable <module_0> .. <module_n>' disables logs in
    specified modules (all if no modules specified).
    enable :'log enable <level> <module_0> ... <module_n>' enables logs
    up to given level in specified modules (all if no modules
    specified).
    go :Resume logging
    halt :Halt logging
    list_backends :Lists logger backends.
    status :Logger status
    mem :Logger memory usage


举例演示如下：

- 将simple模块的日志过滤等级设置为INFO：
  
  *log enable inf simple*
  
- 将simple模块的日志过滤等级设置为DEBUG：
  
  *log enable dbg simple*
  
- 关闭simple模块的日志：
  
  *log disable simple*
  
- 查看日志系统内存使用情况：
  
  *log mem*
  
- 查看日志各模块的过滤状态：
  
  *log status*
  
  显示如下
  
.. code::

    uart:~$ log status
    module_name                              | current | built-in
    ----------------------------------------------------------
    esp_timer                                | inf     | inf
    flash_esp32                              | inf     | dbg
    fs_nvs                                   | inf     | dbg
    gpio_esp32                               | inf     | inf
    i2c                                      | inf     | inf
    i2c_esp32                                | inf     | inf
    i2c_shell                                | inf     | inf
    intc_esp32c3                             | inf     | inf
    log                                      | inf     | inf
    logging_sample                           | wrn     | inf
    module_inst                              | inf     | dbg
    module_inst.inst0                        | inf     | inf
    module_inst.inst1                        | inf     | inf
    module_inst.inst2                        | inf     | inf
    os                                       | inf     | inf
    shell.shell_uart                         | inf     | inf
    shell_uart                               | inf     | inf
    simple                                   | inf     | inf

这里current表示为当前设置的过滤等级，build-in为编译时设置的过滤等级。过滤等级上current不能低于build-in。
   
- 中断所有后端日志输出：
  
  *log halt*
  
- 恢复所有后端日志输出：
  
  *log resume*

常用配置项
==========

消息输出模式
~~~~~~~~~~~~~

\ ``subsys/logging/Kconfig.mode`` 

消息输出模式决定了日志系统的效能和消息的输出方式，总计有4总模式，只能选择一个不能同时存在

- \ ``CONFIG_LOG_MODE_DEFERRED=y``  延迟输出模式，是日志系统默认的输出方式。前端异步发送，耗时的处理被延迟到日志系统内的线程中处理，这种方式对应用的影响最小。
  
- \ ``CONFIG_LOG_MODE_IMMEDIATE=y``  立即输出模式，就在当前的上下文中输出消息。前端调用后端同步发送，直接影响调用者的效能。
  
- \ ``CONFIG_LOG_MODE_MINIMAL=y``  最小实现模式，日志直接通过 ``printk`` 同步输出。没有标签，时间戳，不支持颜色和异步延迟，不支持运行时过滤。只有非常简单的日志等级标识，示例
  
.. code::
    
  > E: Error message example.
  > W: Warning message example.
  > I: Info message example.
  
- \ ``CONFIG_LOG_FRONTEND=y``  输出被重定向到自定前端。
  

过滤
~~~~~~~~~~~~~

\ ``subsys/logging/Kconfig.filtering`` 

过滤的配置决定日志系统过滤器的特性，以下所有配置项的范围都为

- \ ``CONFILG_<MODULE>_LOG_LEVEL`` ：模块的过滤等级，默认为3。
  
- \ ``CONFIG_LOG_DEFAULT_LEVEL`` ：默认的过滤等级，默认为3。
  

日志处理
~~~~~~~~~~~~~

\ ``subsys/logging/Kconfig.processing`` 

- \ ``CONFIG_LOG_PRINTK=y`` ： ``printk`` 的输出被重定向到日志系统，默认为 ``n`` 。
- \ ``CONFIG_LOG_PRINTK_MAX_STRING_LENGTH=256`` ： ``printk`` 可处理的最大字符串长度，超过将被截断。默认为128字节，使用的是堆栈空间。
- \ ``CONFIG_LOG_MODE_OVERFLOW=y`` ：日志消息溢出处理方式， ``y`` 表示丢掉老的保留新的， ``n`` 表示丢掉新的。默认为 ``y`` 。
- \ ``CONFIG_LOG_BLOCK_IN_THREAD=y`` ：前端写消息可等待，如果等待不到再做溢出处理。搭配 ``CONFIG_LOG_BLOCK_IN_THREAD_TIMEOUT_MS`` 进行等待时间设置，默认为 ``n`` 不等待。
- \ ``CONFIG_LOG_BLOCK_IN_THREAD_TIMEOUT_MS`` 前端写消息等待时间，可设置范围 ``-1~10000`` ，单位为毫秒。 ``-1`` 表示永远等待， ``0`` 表示不等待。默认为 ``1000`` ，等待1秒。
- \ ``CONFIG_LOG_PROCESS_TRIGGER_THRESHOLD=3`` 消息输出的阈值，当日志系统的缓存的消息超过该阈值时才会向后端输出，默认为 ``10`` 。该配置和 ``CONFIG_LOG_PROCESS_THREAD_SLEEP_MS`` 只要有一个满足就会使用后端输出。
- \ ``CONFIG_LOG_PROCESS_THREAD=y`` ：日志系统自带线程用于延迟输出，默认为 ``y`` 。当设置为 ``n`` 时，需要用户调用 ``log_process`` 进行消息输出处理。
- \ ``CONFIG_LOG_PROCESS_THREAD_STARTUP_DELAY_MS=100`` ：日志系统自带线程创建后延迟启动时间，默认为 ``0`` 不延迟。
- \ ``CONFIG_LOG_PROCESS_THREAD_SLEEP_MS=10`` ：日志系统自带线程唤醒周期，按该间隔进行日志输出处理，该事件越短显示就越从前端输入到后端输出的间隔就越短。默认为 ``1000`` 也就是1秒。当希望LOG输出比较即时时可以缩段该时间。
- \ ``CONFIG_LOG_BUFFER_SIZE`` ：日志系统消息缓存大小，可配置范围 ``128~65536`` ，默认为 ``1024`` ，该项越大越不容易丢日志。
- \ ``CONFIG_LOG_TRACE_SHORT_TIMESTAMP=y``  使用时间戳，默认为 ``y`` 时间戳的长度为24bit。配置为 ``n`` 时表示不使用时间戳。
- \ ``CONFIG_LOG_TIMESTAMP_64BIT=y``  ：默认为 ``n`` ，配置为 ``y`` 时使用64bit时间戳。
- \ ``CONFIG_LOG_SPEED=y`` ：默认为 ``n`，配置为 ``y`` 时日志系统牺牲空间换取执行时间的提升。

格式
~~~~~~~~~~~~~

\ ``subsys/logging/Kconfig.processing`` 

- \ ``LOG_FUNC_NAME_PREFIX_*`` ：日志中加入函数名标签，只对 ``LOG_*`` 有效，对 ``LOG_HEXDUMP_*`` 无效。
  - \ ``CONFIG_LOG_FUNC_NAME_PREFIX_ERR=y``  在ERROR等级消息中加入函数名，默认为 ``n`` 。
  - \ ``CONFIG_LOG_FUNC_NAME_PREFIX_WRN=y``  在WARNING等级消息中加入函数名，默认为 ``n`` 。
  - \ ``CONFIG_LOG_FUNC_NAME_PREFIX_INF=y``  在INF等级消息中加入函数名，默认为 ``n`` 。
  - \ ``CONFIG_LOG_FUNC_NAME_PREFIX_DBG=y``  在DEBUG等级消息中加入函数名，默认为 ``y`` 。
- \ ``CONFIG_LOG_BACKEND_SHOW_COLOR=y`` ：后端日志消息显示颜色，默认为 ``y`` 要显示。ERROR等级的显示为红色，WRANING等级的显示为黄色。颜色对 ``shell_log_backend`` 无效。
- \ ``CONFIG_LOG_INFO_COLOR_GREEN=y`` ：后端INFO等级消息显示为绿色，默认为不显示颜色。

后端
~~~~~~~~~~~~~

\ ``subsys/logging/Kconfig.backend`` 
在没有配置 ``CONFIG_SHELL=y`` 的情况下默认开启 ``CONFIG_SHELL_LOG_BACKEND`` 使用shell作为LOG的后端。
- \ ``CONFIG_LOG_BACKEND_UART=y``  启用串口后端，默认为 ``n`` 启用。当没有开启Shell时默认使用UART作为LOG后端。
- \ ``CONFIG_LOG_BACKEND_NET`` 启用网络后端，默认为 ``n`` 不启用。
- \ ``CONFIG_LOG_BACKEND_FS`` 启用文件系统 后端，默认为 ``n`` 不启用。


参考
=====

https://docs.zephyrproject.org/latest/services/logging/index.html
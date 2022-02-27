.. _shell_shell_usage:

Zephyr Shell系统使用指南
#########################

Zephyr的shell系统提供一个类Unix shell界面，用户可以自定义shell命令和处理函数，通过shell界面调用这些处理函数。shell系统可以作为软件系统普通功能的人机界面，也可以通过shell系统埋入调试命令，方便进行软件的调试和分析。

Zephyr的shell系统提供以下功能：

- 支持多实例：多个后端可以开多个shell。

- 与log系统合作共存。

- 支持静态和动态命令。

- 支持字典命令。

- 支持自动补全。

- 内建命令。

- 查看历史命令。

- 支持在线编辑修改命令，支持多行命令。

- 支持ANSI转义码。

- 支持通配符。

- 支持meta key。

- 支持`getopt`和`getopt_long`。

- 可配置剪裁。


Shell系统中一些功能对内存和闪存有较大需求，可以通过禁用不需要的功能减少对内存和闪存的使用。配置`CONFIG_SHELL_MINIMAL=y`可以极大的降低存储的使用，在此基础上有选择地只启用想要的功能，以达到存储的最优使用。

shell命令添加方法
==================

实现命令处理函数
-------------------

shell命令处理函数的参数形式是固定的

.. code:: c

   static int cmd_handler(const struct shell *shell, size_t argc, char **argv)

-  **shell** – **[in]** shell的实例后端。

-  **argc** – **[in]** 参数的个数，命令会被计入参数个数中。

-  **argv** – **[in]** 参数字符串指针，所有参数都会以字符串的形式送到命令函数.

例如执行shell命令\ *shell_sample 1 2 3*\ ，命令函数得到的参数如下:

::

   argc=4
   argv[0]="sample"
   argv[1]="1"
   argv[2]="2"
   argv[3]="3"

实例：这里实现三个命令函数

.. code:: c

   static int cmd_info_board(const struct shell *sh, size_t argc, char **argv)
   {
       ARG_UNUSED(argc);
       ARG_UNUSED(argv);

       shell_print(sh, CONFIG_BOARD);

       return 0;
   }

   static int cmd_info_version(const struct shell *sh, size_t argc, char **argv)
   {
       ARG_UNUSED(argc);
       ARG_UNUSED(argv);

       shell_print(sh, "Zephyr version %s", KERNEL_VERSION_STRING);

       return 0;
   }


   static int cmd_shell_help(const struct shell *sh, size_t argc, char **argv)
   {
       shell_print(sh, "show help: %d", argc);
       if(argc == 1){
           shell_help(sh);
           return SHELL_CMD_HELP_PRINTED;
       }

       for(size_t i=0; i< argc; i++){
           shell_print(sh, "check arg %d: %s", i, argv[i]);
       }

       return 0;
   }

注册命令
----------

使用下面三个宏注册静态的shell命令，一旦注册后无法在运行时修改注册的shell命令
**创建命令**

.. code:: c

   SHELL_CMD(_syntax, _subcmd, _help, _handler)

-  \**_syntax\ ** – **\ [in]** 命令符号。

-  \**_subcmd\ ** – **\ [in]** 指向子命令，为空表示没有子命令。子命令可以再嵌入子命令。

-  \**_help\ ** – **\ [in]** 命令帮助信息。

-  \**_handler\ ** – **\ [in]** 命令函数，，为空表示没有命令函数。

**创建子命令集**

.. code:: c

   SHELL_STATIC_SUBCMD_SET_CREATE(name, ...)

-  **name** – **[in]** 子命令集名.

-  **…** – **[in]** 由多个\ ``SHELL_CMD``\ 或\ ``SHELL_ARG_CMD``\ 组成的，并由\ ``SHELL_SUBCMD_SET_END``\ 结束。

**注册命令**

.. code:: c

   SHELL_CMD_REGISTER(syntax, subcmd, help, handler)

-  **syntax** – **[in]** 命令符号

-  **subcmd** – **[in]** 指向子命令，为空表示没有子命令。

-  **help** – **[in]** 命令帮助信息。

-  **handler** – **[in]** 命令函数，，为空表示没有命令函数。

实例：这里使用上面三个宏注册命令

.. code:: c

   /* SHELL_CMD 注册两个子命令， board和version，执行时会调用cmd_info_board和cmd_info_version函数
      SHELL_STATIC_SUBCMD_SET_CREATE 将子命令组装成子命令集subinfo
      SHELL_SUBCMD_SET_END表示子命令集的结束
    */
   SHELL_STATIC_SUBCMD_SET_CREATE(subinfo,
       SHELL_CMD(board, NULL, "Show board command.", cmd_info_board),
       SHELL_CMD(version, NULL, "Show info command.", cmd_info_version),
       SHELL_SUBCMD_SET_END /* Array terminated. */
   );

   /* 注册一个根命令shell_sample，执行根命令shell_sample时会调用cmd_shell_help
       shell_sample的子命令集为
    */
   SHELL_CMD_REGISTER(shell_sample, &subinfo, "Sample commands", cmd_shell_help);

执行命令
--------

上面实例中注册了根命令 *shell_sample*
和其子命令集subinfo，其命令为树机构

::

   shell_sample
       ├── board
       └── version

在shell中可以执行的命令如下: 执行\ *shell_sample*
会调用\ ``cmd_shell_help``\ 显示出帮助信息和参数个数 执行\ *shell_sample
board* 会调用\ ``cmd_info_board``\ 显示出开发板的字符串
执行\ *shell_sample version*
会调用\ ``cmd_info_version``\ 显示zephyr的版本

命令参数
---------

使用\ ``SHELL_CMD_REGISTER``\ 和\ ``SHELL_CMD``\ 注册的命令，shell系统并不会为其检查参数个数。zephyr提供了另外两个宏注册命令，在注册时可以指定参数个数。在执行shell命令时shell系统会根据指定的参数个数进行检查，如果不匹配将不执行命令函数，并进行错误提示。
**创建带参数命令**

.. code:: c

   SHELL_CMD_ARG(syntax, subcmd, help, handler, mand, opt)

-  **syntax** – **[in]** 命令符号。

-  **subcmd** – **[in]** 指向子命令，为空表示没有子命令。子命令可以再嵌入子命令。

-  **help** – **[in]** Pointer to a command help string.

-  **handler** – **[in]** 命令帮助信息。

-  **mand** – **[in]** 必选参数个数，参数个数包含命令本身。

-  **opt** – **[in]** 可选参数个数。

.. code:: c

   SHELL_CMD_ARG_REGISTER(syntax, subcmd, help, handler, mandatory, optional)

-  **syntax** – **[in]** 命令符号

-  **subcmd** – **[in]** 指向子命令，为空表示没有子命令。

-  **help** – **[in]** 命令帮助信息。

-  **handler** – **[in]** 命令函数，，为空表示没有命令函数。

-  **mandatory** – **[in]** 必选参数个数，参数个数包含命令本身。

-  **optional** – **[in]** 可选参数个数。

注意无论由那种方式定义的shell命令，shell系统都会创建\ ``argc``\ 和\ ``argv``\ 并交由注册的命令函数处理。但由\ ``SHELL_CMD_ARG``\ 定义的命令会指定必选参数数量\ ``mandatory``\ 和可选参数数量\ ``optional``\ ，实际通过shell命令输入的参数数量(包含命令本身)要满足：

::

   mandatory <= argc <= mandatory + optional

当不满足时，shell系统会检查参数数量出来并做如下提示

.. code:: c

   shell_sample_args: wrong parameter count

当\ ``mandatory``\ 和\ ``optional``\ 均被设置为0时，Shell系统不会对参数数量进行检查。
以下实例

.. code:: c

   SHELL_CMD_ARG_REGISTER(shell_sample_args, NULL, "Sample arg commands with handle", cmd_shell_help, 3, 4);

表示\ *shell_sample_args*\ 必须有3个参数（包含shell_sample_args），但总计不能超过7个，例如
*shell_sample_args 0* 非法 *shell_sample_args 0 1* 合法
*shell_sample_args 0 1 2 3* 合法 *shell_sample_args 0 1 2 3 4 5 6* 非法


动态命令
==========


动态命令使用方法
----------------

动态子命令由\ ``SHELL_DYNAMIC_CMD_CREATE``\ 生成

.. code:: c

   SHELL_DYNAMIC_CMD_CREATE(name, get)

-  **name** – **[in]** 动态命令入口.

-  **get** – **[in]** 一个\ ``typedef void (*shell_dynamic_get)(size_t idx, struct shell_static_entry *entry)``\ 类型的函数，根据\ ``idx``\ 返回不同的\ ``struct shell_static_entry``\ 类型的命令入口参数。

   ``struct shell_static_entry``\ 和\ ``SHELL_CMD_ARG``\ 指定的静态命令格式一样，指定命令符号，帮助信息，子命令入口，命令函数，和参数个数

   .. code:: c

      struct shell_static_args {
          uint8_t mandatory; /*!< 必选参数个数. */
          uint8_t optional;  /*!< 可选参数个数. */
      };

   .. code:: c

      struct shell_static_entry {
          const char *syntax;            /*!< 命令符号字符串. */
          const char *help;            /*!< 帮助信息字符串. */
          const struct shell_cmd_entry *subcmd;    /*!< 子命令入口. */
          shell_cmd_handler handler;        /*!< 命令函数. */
          struct shell_static_args args;        /*!< 命令参数个数. */
      };

   从上面看到get函数返回的shell命令入口和静态命令对应的参数一模一样,


动态命令示例
------------

如下代码片段演示了如何添加一个动态子命令:

1. 准备一个\ ``typedef void (*shell_dynamic_get)(size_t idx, struct shell_static_entry *entry)``\ 类型的动态命令获取函数：

   .. code:: c

      static void dynamic_cmd_get(size_t idx, struct shell_static_entry *entry)
      {
          if(idx<dynamic_cmd_num){
              entry->syntax = cmd_name_flag? change_name[idx]:dynamic_entrys[idx].syntax;
              entry->handler  = dynamic_entrys[idx].handler;
              entry->subcmd = dynamic_entrys[idx].subcmd;
              entry->help = dynamic_entrys[idx].help;
              entry->args.mandatory = dynamic_entrys[idx].args.mandatory;
              entry->args.optional = dynamic_entrys[idx].args.optional;
          }else{
              entry->syntax = NULL;
          }
      }

2. 生成为动态子命令\ ``dynamic_set``

   .. code:: c

      SHELL_DYNAMIC_CMD_CREATE(dynamic_set, dynamic_cmd_get);

3. 将\ ``dynamic_set``\ 注册到根命令中

   .. code:: c

      SHELL_CMD_REGISTER(shell_dynamic, &dynamic_set,
                 "Sample dynamic command usage.", NULL);

根命令\ *shell_dynamic*\ 的子命令集为\ ``dynamic_set``\ ，子命令集中有哪些子命令是由\ ``dynamic_cmd_get``\ 的输出决定。shell系统在遍历根命令\ *shell_dynamic*\ 的子命令时，传入参数\ ``idx``\ 从0开始每次加一的循环调用\ ``dynamic_cmd_get``\ ，直到输出的\ ``entry->syntax``\ 为空。在上面的示例代码中动态子命令的个数由\ ``dynamic_cmd_num``\ 决定，默认情况下就是\ ``dynamic_entrys[]``\ 中的命令入口数量：

.. code:: c

   /* 动态命令数组 */
   static struct shell_static_entry dynamic_entrys[]=
   {
       /* 改变动态命令总数 */
       {
           .syntax = "total",
           .handler = cmd_dynamic_total,
           .subcmd = NULL,
           .help = "Set total cmd number, must more than 1.",
           .args.mandatory = 0,
           .args.optional = 0,
       },

       /* 使用默认子命令名 */
       {
           .syntax = "org_name",
           .handler = cmd_dynamic_org_name,
           .subcmd = NULL,
           .help = "Change to org cmd name.",
           .args.mandatory = 0,
           .args.optional = 0,
       },

       /* 使用新的子命令名 */
       {
           .syntax = "new_name",
           .handler = cmd_dynamic_new_name,
           .subcmd = NULL,
           .help = "Change to new cmd name.",
           .args.mandatory = 0,
           .args.optional = 0,
       },

       /* 动态子命令的子命令演示 */
       {
           .syntax = "subcmd",
           .handler = NULL,
           .subcmd = &shell_sample,
           .help = "Show dynamic sub cmd.",
           .args.mandatory = 0,
           .args.optional = 0,
       },
       {
           .syntax = "cmd1",
           .handler = NULL,
           .subcmd = NULL,
           .help = "Show dynamic command cmd1.",
           .args.mandatory = 0,
           .args.optional = 0,
       },
       {
           .syntax = "cmd2",
           .handler = NULL,
           .subcmd = NULL,
           .help = "Show dynamic command cmd2.",
           .args.mandatory = 0,
           .args.optional = 0,
       },
       {
           .syntax = NULL,
       }
   };

   static uint32_t dynamic_cmd_num = sizeof(dynamic_entrys)/sizeof(struct shell_static_entry);

开机后在shell中输入\ ``shell_dynamic``\ 后按tab后可以看到所有的子命令

.. code:: c

   uart:~$ shell_dynamic
     total     org_name  new_name  subcmd    cmd1      cmd2

可以在运行时改变\ ``dynamic_cmd_num``\ 的大小达到改变\ *shell_dynamic*\ 的子命令数量，例如\ *total*\ 子命令就可以达到这一目的，它对应的命令函数如下：

.. code:: c

   static int cmd_dynamic_total(const struct shell *sh, size_t argc, char **argv)
   {
       if(argc < 2){
           shell_help(sh);
           return SHELL_CMD_HELP_PRINTED;
       }

       uint32_t num = (uint32_t)atoi(argv[1]);
       if(num<1 && num > sizeof(dynamic_entrys)/sizeof(struct shell_static_entry)){
           shell_error(sh, "total set fail, must in 1~%d", sizeof(dynamic_entrys)/sizeof(struct shell_static_entry));
           return -ENOEXEC;
       }

       /* 改变动态命令的数量 */
       shell_print(sh, "set total cmd num %d", num);
       dynamic_cmd_num = num;

       return 0;
   }

如果执行shell命令\ *shell_dynamic total
1*\ ，\ ``cmd_dynamic_total``\ 被调用将\ ``dynamic_cmd_num``\ 修改为3，此时在shell中输入\ *shell_dynamic*\ 后按tab后就只能看到只剩3个子命令：

::

   uart:~$ shell_dynamic
     total     org_name  new_name

也可以在运行时通过改变\ ``cmd_name_flag``\ 变量的值，让动态子命令的\ ``syntax``\ 发生改变，达到动态改变动态子命令符号的目的。例如执行\ *shell_dynamic
new_name*\ 后\ ``cmd_name_flag``\ 被修改为1，\ ``dynamic_cmd_get``\ 输出的\ ``syntax``\ 将使用\ ``change_name``\ 的内容

.. code:: c

   char * change_name[] = {
       "total_new",
       "org_name_new",
       "new_name_new",
       "cmd1_new",
       "cmd2_new",
       "cmd3_new",
   };

此时在shell中输入\ *shell_dynamic*\ 后按tab后，看到的就是新的子命令名称

::

   uart:~$ shell_dynamic
     total_new     org_name_new  new_name_new

当然也可以修改\ ``dynamic_entrys``\ 的内容达到添加和删除动态子命令的目的，或者是修改其中的子命令函数达到修改子命令功能的目的。



内置命令
========

shell系统自带内置命令，自带命令可以通过配置项进行配置是否启用来优化shell的空间占用，默认情况下\ ``CONFIG_SHELL_CMDS=y``\ 开启了部分内置命令，将其设置为\ ``n``\ 可以关闭内置命令。

Shell的内置命令列表：

-  **clear** ：清屏

-  **help**\ ：显示shell所有根命令及帮助信息

-  **history**\ ：显示最近执行了的命令

-  **resize**\ ：改变终端尺寸。当执行较长命令后，为保证多行显示和←, →, End, Home正常，需要执行该命令重设终端尺寸，目前只有UART后端支持该命令。

   -  *resize* 命令默认是开启的\ ``CONFIG_SHELL_CMDS_RESIZE=y``

   -  默认情况下执行\ *resize*\ 后终端的尺寸为80x24，可以通过\ ``CONFIG_SHELL_DEFAULT_TERMINAL_WIDTH=80``\ 和\ ``CONFIG_SHELL_DEFAULT_TERMINAL_HEIGHT=24``\ 配置改变

   -  *resize*\ 有一个\ *default*\ 子命令，无论默认配置为多少执行\ *resize
      default*\ 后就会把终端大小设置为80X24。

-  **select**\ ：设置根，通过alt+r可以退回到主根。

   例如主根下按tab可以看到所有的根命令：

   ::

      uart:~$
        clear                 device                devmem
        help                  history               kernel
        log                   logging               nrf_clock_control
        resize                select                shell
        shell_dict            shell_dynamic         shell_sample
        shell_sample_args     shell_sample_handler  shell_sample_null
        shell_sample_sub

   当执行\ *select
   shell_sample*\ 后，在主根下按tab就只能看到\ *shell_sample*\ 的子命令：

   ::

      uart:~$
        info     subinfo  arginfo

   再按看到下面提示表示退回到默认的主根

   ::

      Restored default root commands

   *select*\ 命令默认是关闭的，需要配置\ ``CONFIG_SHELL_CMDS_SELECT=y``\ 开启

       

-  **shell**\ ：用于设置shell终端的属性，有如下子命令

   -  *shell backspace_mode
      backspace*\ ：设置Backspace按键为backspace模式，按该按键后不会删除已输入的内容

   -  *shell backspace_mode
      delete*\ ：设置Backspace按键为delete模式，按该按键删除已输入的内容

   -  *shell color off*\ ：关闭shell终端颜色

   -  *shell color on*\ ：开启shell终端颜色

   -  *shell echo
      off*\ ：关闭shell回显，输入的命令不会被回显。需要依赖使用的终端软件支持回显。在终端软件必须回显的情况下可以用该命令关闭shell系统的回显。

   -  *shell echo on*\ ：开启shell回显，输入的命令被回显。

   -  *shell stats reset*\ ：清除log系统丢消息的统计信息。

   -  *shell stats show*\ ：显示log系统丢的消息。




命令行特性
==========


Zephyr的shell系统提供一个类Unix shell界面，通过该命令行界面用户可以操作Zephyr或者用户自己定义的shell命令。Zephyr的shell系统提供了一系列命令行特性方便操作shell命令。

文本编辑按键支持
--------------------

左右移动光标：←, → 删除光标所在字符：Backspace, Delete
移动光标到行尾/首：End, Home 切换插入/覆盖模式：Insert

自动补全
----------

默认\ ``CONFIG_SHELL_TAB=y``\ 开启了tab支持

Tab按键支持以下特性：

-  提示有效命令

   当按下tab时，会自动提示出所有有效的命令。例如敲入\ *she*\ 后按下tab会将以\ *she*\ 开头的命令都提示出来

   ::

      uart:~$ she
        shell                 shell_dict            shell_dynamic
        shell_sample          shell_sample_args     shell_sample_handler
        shell_sample_null     shell_sample_sub

-  自动补全

   默认\ ``CONFIG_SHELL_TAB_AUTOCOMPLETION=y``\ 开启了自动补全，当提示命令只有一条时就会自动补全被执行。例如敲入\ *shell_sample_s*
   后按下tab会将命令自动补全到输入位置

   ::

      uart:~$ shell_sample_sub

历史命令
----------

默认\ ``CONFIG_SHELL_HISTORY=y``\ 开启了命令历史记录。通过执行 *history*
命令可以查看历史执行过的命令。以通过↑ 和↓ 按键切换选择已经执行过的命令。当启用meta按键后也可以通过 Ctrl + n 和 Ctrl + p来切换选择。

通配符
----------

默认\ ``CONFIG_SHELL_WILDCARD=y``\ 开启了通配符支持，shell支持两个通配符：

-  \* ：匹配字符串

-  ? ：匹配单个字符

MetaKey
----------

默认情况下\ ``CONFIG_SHELL_METAKEYS=y``\ 开启了metakey的支持。shell支持的metakey和作用如下表

+-----------+---------------------------------------------------------+
| Meta keys | Action                                                  |
+===========+=========================================================+
| Ctrl + a  | 移动光标到行首，等同于Home                              |
+-----------+---------------------------------------------------------+
| Ctrl + b  | 将光标向左移动一个字符，等同于←                         |
+-----------+---------------------------------------------------------+
| Ctrl + c  | 放弃当前行输入的内容，另外新开                          |
|           | 一行用于输入命令。类似于回Enter但不执行已经输入了的命令 |
+-----------+---------------------------------------------------------+
| Ctrl + d  | 删除光标下的字符，等同于Delete                          |
+-----------+---------------------------------------------------------+
| Ctrl + e  | 移动光标到行尾，等同于End                               |
+-----------+---------------------------------------------------------+
| Ctrl + f  | 将光标向右移动一个字符，等同于→                         |
+-----------+---------------------------------------------------------+
| Ctrl + k  | 删除从光标到行尾的所有字符                              |
+-----------+---------------------------------------------------------+
| Ctrl + l  | 保留当前正在输入的命令，清除屏幕其它的内容。            |
+-----------+---------------------------------------------------------+
| Ctrl + n  | 切换到上一个历史执行的命令                              |
+-----------+---------------------------------------------------------+
| Ctrl + p  | 切换到下一个历史执行的命令                              |
+-----------+---------------------------------------------------------+
| Ctrl + u  | 清除当前正在输入的命令                                  |
+-----------+---------------------------------------------------------+
| Ctrl + w  | 删除光标侧的一个单词                                    |
+-----------+---------------------------------------------------------+
| Alt + b   | 移动光标到前一个词                                      |
+-----------+---------------------------------------------------------+
| Alt + f   | 移动光标到后一个词                                      |
+-----------+---------------------------------------------------------+



参考
=====
https://docs.zephyrproject.org/latest/services/shell/index.html
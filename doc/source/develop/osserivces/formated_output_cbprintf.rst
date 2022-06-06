.. _osservices_formated_output_cbprintf:

CBPRINTF
##############################

本文说明Zephyr格式化输出的配置和限制。

基本概念
========

常规C99标准libc库中中的\*printf实现是可以支持流式输出设备，并且也能提供内部缓存。但对于嵌入式设备来说可能都不要求持这两种形式以达到降低使用资源的目的。因此zephyr专门设计了cbprintf用来转换C99指定的格式化字符串和参数。cbprintf可以支持所有的C99格式规范。


同步回调输出
~~~~~~~~~~~~

cbprintf通过回调机制,对格式化字符串和参数进行一边解析一边输出，避免使用大量缓存。
printk/printf/shell_print使用的同步机制,也就是使用下面的API

.. code:: c

  int cbprintf(cbprintf_cb out, void *ctx, const char *format, ...);
  int cbvprintf(cbprintf_cb out, void *ctx, const char *format, va_list ap);


异步cbprintf包
~~~~~~~~~~~~~~~

在有些情况下，格式化会被推迟，例如log系统就是在API被调用的时候打包加入到buffer，core task获取包后用下列标准化函数处理。在这种情况下，格式化字符串和参数被转换成了一个独立的包。包的主要内容类似于va_list堆栈框架，因此标准的格式化函数被用来处理一个包。

.. code:: c

  int cbprintf_package(void *packaged, size_t len, uint32_t flags, const char *format, ...);
  int cbvprintf_package(void *packaged, size_t len, uint32_t flags, const char *format, va_list ap);
  CBPRINTF_STATIC_PACKAGE(packaged, inlen, outlen, align_offset, flags, ...)


当格式为%s时，对应的参数是一个指向有字符串的指针，默认情况下包只会有该指针而不包含字符串内容，使用下面API可以将字符串的内容加入到包中

.. code:: c

  int cbprintf_fsc_package(void *in_packaged, size_t in_len, void *packaged, size_t len);

下面API可以将包以同步的方式通过回调函数一边解析一边输出

.. code:: c

  int cbpprintf(cbprintf_cb out, void *ctx, void *packaged);


代码及位置
==========

格式化字符串解析和包生成解析是cbprintf的核心实现内容，本文无意进行分析，只对其文件位置和作用进行说明：
\ ``include\sys\cbprintf.h``  cbprintf.h头文件提供所有外部函数和宏
\ ``lib\os\cbprintf_packaged.c`` 实现了cbprintf包的处理
\ ``lib\os\cbprintf.c`` 格式化输出函数封装实现，通过配置文件在nano和complete当作二选一
\ ``lib\os\cbprintf_nano.c``  格式化输出函数的nano实现，可以优化对资源的需求，但大多数格式都不被支持。
\ ``lib\os\cbprintf_complete.c`` 格式化输出函数的完整实现，默认情况下会使用该实现。
\ ``lib\os\Kconfig.cbprintf`` cbprintf的配置文件

配置及影响
==========

CBORINTF实现
~~~~~~~~~~~~

决定使用Complete还是nano实现，在risc-v上nano的二进制代码比Complete小1.5KByte左右， ``CONFIG_CBPRINTF_COMPLETE`` 和 ``CONFIG_CBPRINTF_NANO`` 这两个选项只能二选一配置。要分析二者的差异先要看格式化字符串的构成：
\ ``%[flags][field width][.precision][length]type`` 。
例如

.. code:: c

  printk("%04lld", a);   // flags: 0, filed width: 4, length: ll, type: d
  printk("%.3s", "abcdefg"); // .precision: 3 , type: s

这里对比说明两种配置对格式化字符串的支持情况。

flags
^^^^^^^

符号，可以包含下列一个或多个：

* \+ : 显示数值符号
* 空格 : 用空格补齐数值符号
* \- : 左对齐，默认是右对齐
* 0 : 宽度不足补0
* \# : 添加进制数符号(0x, o)，显示浮点数小数后面的0

\ ``CONFIG_CBPRINTF_COMPLETE`` 和 ``CONFIG_CBPRINTF_NANO`` 都支持。

field width
^^^^^^^^^^^^^^

给出显示数值的最小宽度，典型用于制表输出时填充固定宽度的表目。实际输出字符的个数不足域宽，则根据左对齐或右对齐进行填充。实际输出字符的个数超过域宽并不引起数值截断，而是显示全部。宽度值的前导0被解释为0填充标志；前导的负值被解释为其绝对值，负号解释为左对齐标志。如果域宽值为*，则由对应的函数参数的值为当前域宽。

\ ``CONFIG_CBPRINTF_COMPLETE`` 和 ``CONFIG_CBPRINTF_NANO`` 都支持。

\.precision
^^^^^^^^^^^^^^

指明输出的最大长度，依赖于特定的格式化类型。对于d、i、u、x、o的整型数值，是指最小数字位数，不足的位要在左侧补0，如果超过也不截断，缺省值为1。对于a,A,e,E,f,F的浮点数值，是指小数点右边显示的数字位数，必要时四舍五入或补0；缺省值为6。对于g,G的浮点数值，是指有效数字的最大位数；缺省值为6。对于s的字符串类型，是指输出的字节的上限，超出限制的其它字符将被截断。如果域宽为*，则由对应的函数参数的值为当前域宽。如果仅给出了小数点，则域宽为0。

\ ``CONFIG_CBPRINTF_COMPLETE`` 和 ``CONFIG_CBPRINTF_NANO`` 都支持。

length
^^^^^^^^^^^^^^

指出浮点型参数或整型参数的长度。

* hh : 期待一个从char提升的int尺寸的参数
* h : 期待一个从short提升的int尺寸的参数
* l : 期待一个long尺寸的参数(整数)/期待一个double尺寸的参数(浮点)
* ll : 期待一个long long尺寸的参数
* L : 期待一个long double尺寸的参数
* z : 期待一个size_t尺寸的参数
* j : 期待一个intmax_t尺寸的参数
* t : 待一个ptrdiff_t尺寸的参数

\ ``CONFIG_CBPRINTF_NANO``  不支持L,j,t，其它都支持。
\ ``CONFIG_CBPRINTF_COMPLETE`` 不支持L，其它都支持。


type
^^^^^^^^^^^^^^

转换说明

* d, i : 有符号十进制数值int
* u : 十进制unsigned int
* f, F : double型输出10进制定点表示
* e, E : double值，输出形式为10进制指数
* g, G : double型数值，精度定义为全部有效数字位数
* x, X : 16进制unsigned int
* o : 8进制unsigned int
* c : 字符
* s : 字符串
* p : void*指针
* a,A : double型的16进制表示
* n : 不输出字符，但是把已经成功输出的字符个数写入对应的整型指针参数所指的变量

\ ``CONFIG_CBPRINTF_NANO``  不支持f,F,e,E,g,G,a,A,n，其它都支持。
\ ``CONFIG_CBPRINTF_COMPLETE`` 配合以下配置支持所有以上type:
* \ ``CBPRINTF_FP_SUPPORT=y`` 时，支持f,F,e,E,g,G
* \ ``CONFIG_CBPRINTF_FP_A_SUPPORT=y`` 时，支持a,A
* \ ``CONFIG_CBPRINTF_N_SPECIFIER=y`` 时，支持n

当配置`CONFIG_CBPRINTF_FP_ALWAYS_A=y`` 时f,F,e,E,g,G都将以a,A的的形式格式化输出。


整型宽度
~~~~~~~~~~

\ ``CONFIG_CBPRINTF_FULL_INTEGRAL`` 或 ``CONFIG_CBPRINTF_REDUCED_INTEGRAL`` 这两个选项只能二选一配置整型宽度，当配置为 ``CONFIG_CBPRINTF_REDUCED_INTEGRAL=y`` 时整型被限制为32bit，会影响到size_t和intmax_t以及指针的转换。



包配置
~~~~~~~

包的配置只对LOG系统有影响
\ ``CONFIG_CBPRINTF_PACKAGE_LONGDOUBLE``  打包时支持long double类型参数。
\ ``CONFIG_CBPRINTF_STATIC_PACKAGE_CHECK_ALIGNMENT``  用于静态包对齐检查。

产生C库兼容函数
~~~~~~~~~~~~~~~

\ ``CONFIG_CBPRINTF_LIBC_SUBSTS=y`` 用于产生C库兼容函数, 函数名在标准输出函数后面加上cb，这样就使用的是cbprint的格式化程序，而不用使用libc中的。

.. code:: c

  int fprintfcb(FILE *stream, const char *format, ...);
  int vfprintfcb(FILE *stream, const char *format, va_list ap);
  int printfcb(const char *format, ...);
  int vprintfcb(const char *format, va_list ap);
  int snprintfcb(char *str, size_t size, const char *format, ...);
  int vsnprintfcb(char *str, size_t size, const char *format, va_list ap);


Zephyr格式化输出总结说明
=========================

当使用printk/shell_print/minilibc中的\*printf/LOG*进行格式化输出时：
Zephyr在不进行手配置情况下，系统默认配置如下，**不支持浮点数的打印**:

.. code::

  CONFIG_CBPRINTF_COMPLETE=y
  CONFIG_CBPRINTF_FULL_INTEGRAL=y

浮点%f, %e, %g打印需要添加配置:
.. code::

  CBPRINTF_FP_SUPPORT=y

%a支持需要添加配置:
.. code::

  CONFIG_CBPRINTF_FP_A_SUPPORT=y

%n支持需要添加配置:
.. code::

  CONFIG_CBPRINTF_N_SPECIFIER=y

要缩小代码尺寸,注意前面nano的格式化限制:
.. code::

  CONFIG_CBPRINTF_NANO=y

要缩小堆栈占用:
.. code::

  CONFIG_CBPRINTF_REDUCED_INTEGRAL=y



参考
=====

https://docs.zephyrproject.org/3.0.0/reference/misc/formatted_output.html
https://www.dii.uchile.cl/~daespino/files/Iso_C_1999_definition.pdf
https://zh.wikipedia.org/wiki/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2

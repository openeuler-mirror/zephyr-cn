 .. _develop_vscode:


使用vscode进行Zephyr应用开发
##############################

背景
=======

许多使用者在进行Zephyr开发时，一大需求就是希望能够使用图形化的IDE进行开发，如像Keil、IAR、CodeWarrior、Eclipse。其实作为一个相对大型的嵌入式软件平台，Zephyr并没有与之完美
配套的IDE，正如也很难找到与Linux内核开发配套的IDE一样。Zephyr社区中主流进行开发的方式是采用编辑器（vim, sublime)等编辑代码，再在命令行中通过一系列工具，如west、ninja, make
进行编译、调试等。这种方式的优点是一旦熟练掌握后便十分灵活方便，但缺点是相应的学习曲线很长，需求熟悉掌握很多工具，而且有时命令行的效率确实不如图形化的方式，例如调试、浏览代码的场合。

新一代编辑器vscode的快速发展以及其丰富的插件体系，使得采用类似IDE的方式进行zephyr开发成为可能，虽然不能解决所有问题，有时还需要结合命令行去完成部分工作，还是值得一试的。本章主要介绍
如何使用vscode进行Zephyr应用开发，主要完成如下任务：

- **高效的代码浏览**
- **Zephyr应用的编译与构建**
- **Zephyr应用的调试**
- **Zephyr的代码提交（待完成）**

.. note::

    Zephyr构建系统的核心是**west+cmake+python工具**, 采用命令行可以完成所有的开发工作，IDE的图形化开发方式是建立在构建系统之上的“皮”， 皮可以换，可以是eclipse，也可以是vscode，
    但本体不会变化。

准备工作
==========

本章所描述的工作都是在Linux环境下实现，所使用的系统为Debian 11，相关内容经简单修改后同样适用于Windows环境。Windows下的Zephyr开发环境可以参见 :ref:`develop_windows`。

首先需要下载并安装好vscode, 并推荐安装如下vscode扩展插件：

.. csv-table:: vscode插件列表
    :header: "插件名", "用途"
    :widths: 100,100

    "C/C++", "C/C++ IntelliSense, 调试与代码浏览"
    "CMake", "CMake语言支持"
    "kconfig", "kconfig语言支持"
    "Arm Assembly", "arm汇编语言支持"
    "nRF DeviceTree", "Zephyr DeviceTree支持"
    "Trailing Spaces", "高亮行尾部多余空格"

代码浏览
==========

* vscode workspace布局
  建议采用vscode multi-root workspace的方式去布局相应文件结构，可参考如下结构：

::

    <vscode zephyr_workspace>
    ├── zephyr  zephyr-os代码
    ├── modules zephyr modules（按需添加）
    ├── app     自定义zephyr应用（如果采用Zephyr自带的例子可以不需要
    └── build   zephyr应用构建目录

.. note::

    为了避免zephyr的构建污染zephyr的git仓库，同时为了方便管理，建议把build目录单独存放

* C/C++ IntelliSense配置

vscode的C/C++ IntelliSense插件功能强大，但由于整个zephyr代码规模较大，IntelliSense的默认配置无法支撑，所以我们需要
借助 :file:`compile_commands.json` 使得C/C++ IntelliSense能够准确找到相关代码。

由于Zephyr的构建系统是基于CMake的，每次CMake配置之后会生成 :file:`compile_commands.json`，里面包含了本次构建所涉及的文件、编
译器选项。需要注意的的是 :file:`compile_commands.json` 只包含了本次构建相关内容，不相关文件不会包含在里面，因此vscode的C/C++ 
IntelliSense只会覆盖相关文件，例如如果构建过程中没使用网络相关功能，那么 :file:`subsys/net` 下的代码则不会被搜索。

在整个vscode workspace层级（注意不是folder层级）设置C/C++ IntelliSense的 **compileCommands** 属性正确指
向 :file:`compile_commands.json` 即可生效。具体的vscode workspace参考配置如下：

    `zephyr vscode workspace参考配置 <https://gitee.com/openeuler/zephyr-cn/blob/master/vscode/zephyr.code-workspace>`_


由于必须先配置CMake后才会有 :file:`compile_commands.json`，因此需要在build下通过如下方式生成：

* 在build目录下，通过命令行生成：

    .. code-block:: bash

        cmake -DBOARD=<target board>  <path-to-zephyr-application>

* 或者通过，运行 **编译与构建** 章节中所定义的 **Zephyr 配置** 任务进行

编译与构建
===========

Zephyr的编译和构建核心是构建在命令行之上的，要实现vscode和Zephyr构建系统的对接，需要使用vscode的 **task** 机制，即为
vscode配置一系列Zephyr相关的task。由于这些task是和构建相关，因此相关实现放在 :file：`build/.vscode/tasks.json` 中，
参考实现如下：

    `zephyr vscode tasks参考配置 <https://gitee.com/openeuler/zephyr-cn/blob/master/vscode/.vscode/tasks.json>`_

有了如上一系列tasks配置之后，就可以在vscode的 **Run Task** 中选择相应的zephyr任务，如构建，menuconfig, clean等。

调试
=========

Zephyr默认的调试界面的是gdb, 要实现vscode和Zephyr调试界面的对接，需要使用vscode的 **lanuch** 机制，即为
vscode配置一系列Zephyr相关的run。由于这些task是和构建相关，因此相关实现放在 :file：`build/.vscode/launch.json` 中，
参考实现如下：

    `zephyr vscode lauch参考配置 <https://gitee.com/openeuler/zephyr-cn/blob/master/vscode/.vscode/launch.json>`_

有了如上一系列launch配置之后，就可以在vscode中唤起gdb,并通过图形化的方式进行调试，包括单步运行、断点、查看变量等等。

代码提交
==========

参与Zephyr社区的代码提交，核心是掌握git相关技巧。当前还是建议通过命令行的方式进行git操作。
.. _develop_windows:


Windows下开发环境的搭建
##############################


新一代的IoT OS平台相对以往的RTOS平台更加复杂，需要许多不同的工具， Zephyr也是如此。Zephyr当前支持在Windows、Linux、Mac OS下的构建，
其中Linux下的支持最好，按照Zephyr官方文档一步一步操作即可。Zephyr对Windows支持是基于 `Chocolatey <https://chocolatey.org/>`_
包管理器，但对于国内用户而言，可能存在两大问题， 第一Chocolatey包管理器安装需要特殊设置，可能无法安装;第二即使安装好后，由于chocolatey
在国内并没有镜像源，下载软件包的速度会很不稳定。

本章主要介绍一种适合国内用户的替代方案。

准备代码
===========

Zephyr的代码可以分为Zephyr OS和Zephyr modules两大部分，都托管在github上。经过多年的发展，整个Zephyr的代码已经相当庞大，如果对github
的访问不可靠的话，那么在第一次准备代码时会存在问题。有鉴于此，SIG-Zephyr提供了基于Zephyr发行版的代码镜像，以方便国内用户访问：

    * 只包含Zephyr OS最新发行版和lts发行版，可以访问 `镜像 <https://gitee.com/src-openeuler/zephyr>`_
    * 包含了Zephyr OS和Zephyr modules，可以访问 `网盘 <https://pan.baidu.com/s/1v6wk386WMKZ8X8cXSdfUOQ>`_ 提取码: **zeph**
    * 如果需要同步Zephyr OS的git仓库，可以先通过 `gitee镜像 <https://gitee.com/zephyr-rtos/zephyr>`_ 同步，然后再同步到github上
      的仓库
    * Zephyr modules数量较多，但只有部分git仓库在 `gitee镜像 <https://gitee.com/zephyr-rtos/zephyr>`_ 上存在，缺失的部分需要通过其方式
      获得。在实际开发过程中，很少会使用到所有Zephyr modules, 因此可以根据需要，只准备相应的modules。

下载或clone好代码后，一种推荐的文件布局如下：

::

    <顶层目录zephyr_workspace>
    ├── bootloader  bootloader模块，目前只有mcuboot
    ├── modules     zephyr modules顶层目录，其下包含各种modules
    ├── tools       zephyr配套的工具目录
    └── zephyr      zephyr OS核心目录（必要)


工具链
========

* Zephyr SDK

Zephyr Project自身提供了一个包含编译器、QEMU仿真器、openOCD调试等工具在内的SDK(Software Development Kit)， 称之为Zephyr SDK。
通过维护Zephyr SDK，Zephyr Project提供了一个完整的解决方案，无须再安装其他工具。同时，Zephyr SDK与Zephyr OS共同演进发布，并且经过
测试，很好地解决了工具与OS的兼容性问题。一般情况下，最新版本的Zephyr SDK配套最新版本的Zephyr, 用户可以通过如下链接获取Zephyr SDK:

    `Zephyr SDK <https://github.com/zephyrproject-rtos/sdk-ng/releases>`_

Zephyr SDK中的编译器是基于GCC的，包含Zephyr支持的所有处理器架构，因此完整的SDK自解压安装包大于1 GB。用户也可以选择只包含所用到处理器架构
编译器的SDK安装包，大约在100-200 MB之间。

Zephyr SDK是发布在github上， 国内访问速度比较慢且不稳定，SIG-Zephyr也提供了基于百度网盘的最新SDK镜像：

    `最新版Zephyr SDK镜像 <https://pan.baidu.com/s/1v6wk386WMKZ8X8cXSdfUOQ>`_ 提取码: **zeph**


当前Zephyr SDK是基于Yocto构建的，因此只支持Linux环境。Zephyr SDK正计划迁移到基于 `cross-ng <https://crosstool-ng.github.io/>`_ 的构建，
届时为会Windows，Linux和MAC OS构建出相应的工具链。如果有兴趣尝试，可以通过链接获取原型：

    `下一代Zephyr SDK原型 <https://github.com/zephyrproject-rtos/sdk-ng-testing/releases>`_

* GNU ARM Embedded工具链

如前文所述，Zephyr SDK暂不支持Windows环境，需要使用第三方的工具链，对于ARM 32位架构硬件（ARM Cortex M系列， ARM Cortex R系列等）推荐使用
GNU ARM Embedded工具链：

    `Windows GNU ARM Embedded工具链 <https://developer.arm.com/open-source/gnu-toolchain/gnu-rm>`_

假设下载后并安装在 :file:`C:\\gnu_arm_embedded` 下。

主机工具
============

下载好编译器之后，接下来是准备主机工具。Zephyr对主机工具的最小依赖如下：


.. list-table::
   :header-rows: 1

   * - 工具名
     - 最低版本

   * - `CMake <https://cmake.org/>`_
     - 3.20.0

   * - `Python <https://www.python.org/>`_
     - 3.6

   * - `Devicetree compiler <https://www.devicetree.org/>`_
     - 1.4.6

   * - `ninja <https://github.com/ninja-build/ninja/releases/download/v1.10.2/ninja-win.zip>`_
     - 1.10.2 (windows需要)

   * - `gperf <https://www.gnu.org/software/gperf/>`_
     - 3.1

上述工具中，CMAKE、Python和ninja都有windows版本，请下载、安装并添加到Windows的PATH路径。DTC和gperf是gnu工具，并没有
直接的windows发行版本，而是基于cygwin或msys2环境实现Windows下的运行，例如Chocolatey中的gperf和dtc是其实也是基于msys2
环境。

* 安装msys2的pacman包管理器

由于Chocolatey的限制，此处我们使用另一个 `msys2 <https://www.msys2.org>`_ 提供的包管理器pacman来实现gperf和msys2的
安装。基于msys2的pacman管理器，国内有大量的镜像源且安装方便，不存在类似Chocolatey的问题。 此处使用清华源：

    `pacman安装包清华源 <https://mirrors.tuna.tsinghua.edu.cn/help/msys2/>`_

   .. note::

       安装好后，请更新相应的镜像源地址，尽可能使用国内的镜像源以加速访问，如阿里云，清华，中科大等

* 安装dtc和gperf

打开msys2命令行窗口，安装dtc和gperf

    .. code-block:: bash

        pacman -R dtc gperf

   .. warning::

       我们只需要dtc和gperf，不要安装其他个工具， 由于msys2环境的路径问题，cmake, python, ninja可能存在着不
       兼容的问题

* 安装west

Zephyr的开发并不是必须依赖west, 但通过west可以有效简化一些步骤，因此我们推荐安装west。

    .. code-block:: bash

        pip3  install -U west

west的初始化依赖git, 所以请同时安装好Windows git。

* 安装QEMU

体验Zephyr并不需要有实际的开发板，可以通过QEMU来模拟相应的开发板。

    `QEMU 6.2 windows版 <https://qemu.weilnetz.de/w64/2021/qemu-w64-setup-20211215.exe>`_

设置环境变量以及必要配置
=========================

* Windows的PATH变量：需要把python, cmake, ninja以及msys2的 "<msys2安装目录>\usr\bin"添加到Windows PATH变量中。
  设置好之后，打开cmd窗口，如能找python, cmake, ninja, dtc， gperf， git, qemu-system-arm这些命令，则为成功。

* 安装所需要的python库：

.. code-block:: bash

    pip3 install --user -r <zephyr workspace目录>/zephyr/scripts/requirements.txt

* 为GNU ARM Embedded工具链设置环境变量,在Windows增加了两个环境变量:

  - "ZEPHYR_TOOLCHAIN_VARIANT”值为“gnuarmemb”， 告知Zephyr构建系统使用的是GNU ARM Embedded工具链

  - "GNUARMEMB_TOOLCHAIN_PATH"值为:file:`C:\\gnu_arm_embedded`, 即GNU ARM Embedded安装目录

* 初始化Zephyr workspace

.. code-block:: bash

    cd <zephyr workspace目录>
    west init -l zephyr #通过 -l参数，实现workspace本地初始化
    west zephyr-export  #导出CMake FindPackages

编译Zephyr应用
======================

上述所有准备工作完成之后，可以尝试构建Zephyr应用了。

.. code-block:: bash

    cd <zephyr workspace目录>/zephyr
    west build -p auto -b qemu_cortex_m3 samples/philosophers # 编译zephyr应用
    west build -t run # 通过qemu运行zephyr应用

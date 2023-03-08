 .. _develop_vscode:

West（Zephyr的meta tool）
##############################

Zephyr 项目包括一个精致小巧，功能强大的命令行工具：West。 `West <https://github.com/zephyrproject-rtos/west>`_ 在独立的存储库中开发。

West 的内置命令提供了一个多存储库管理系统，其功能源于 Google 的 Repo 工具和 Git 子模块的启发。West 也拥有“可插拔”的特性：用户可以编写自己的West拓展命令，为 West 添加额外的功能。而 Zephyr 则利用这个特性为构建，下载，调试等功能提供便利。

与 git 和 docker 一样，west 的顶级命令使用一些共用选项和要运行的子命令，然后是该子命令的选项和参数：

.. code-block:: bash

    west [common-opts] <command> [opts] <args>


从 West v0.8 之后，也可以像这样运行 West 命令：

.. code-block:: bash

    python3 -m west [common-opts] <command> [opts] <args>


以下文档记录了 West 的 v1.0.y 之后版本的相关内容

.. toctree::
   :maxdepth: 1

   install.rst
   basic.rst

基本概念
##########

本页介绍了 west 的基本概念，并提供进一步阅读的参考。

West 的内置命令允许您在公共工作区目录下处理项目（Git 存储库）。

示例工作区
**********

如果您遵循了 Zephyr 入门指南，您的工作区将如下所示：

.. code-block:: none

    zephyrproject/                 # west 顶级目录
    ├── .west/                     # 标记顶级目录的位置
    │   └── config                 # 每个工作区的本地配置文件
    │
    │   # The manifest repository, never modified by west after creation:
    ├── zephyr/                    # .git/ repo
    │   ├── west.yml               # 清单文件（manifest）
    │   └── [... other files ...]
    │
    │   # west管理的项目：
    ├── modules/
    │   └── lib/
    │       └── zcbor/             # .git/ project
    ├── net-tools/                 # .git/ project
    └── [ ... 其他项目 ...]

.. _west-workspace:

工作区概念
**********

以下是您应该了解的有关上述工作区结构的基本概念。 其他详细信息位于工作区中。

顶级目录
    在上述例子中，顶级目录是名为 zephyrproject 的目录，通常可以在使用 west init 命令初始化工作区的时候设置顶级目录的名字

.west 目录
    顶级目录包含了 :file:`.west` 目录。west使用 :file:`.west` 目录的父目录作为顶级目录。搜索顺序是从当前目录开始。（如果搜索失败则重新从 :envvar:`ZEPHYR_BASE` 这个环境变量所描述的位置重新开始搜索。）

配置文件
    :file:`.west/config` 文件是作用域为当前工作区的配置文件，即局部配置文件

清单仓库（manifest repository）
    每个 west 工作区只包含一个清单仓库，这是一个包含 manifest 文件的 Git 仓库。 清单仓库的路径由局部配置文件中的 *manifest repository* 配置选项给出。
    对于上游 Zephyr，zephyr 是清单存储库，但您可以将 west 配置为使用工作区中的任何 Git 存储库作为清单存储库。 唯一的要求是它包含一个有效的清单文件。 有关其他选项的信息，请参阅支持的拓扑，有关清单文件格式的详细信息，请参阅 West Manifests。

清单文件（manifest file）
    清单文件是一个YAML 文件，列出了在工作区中west需要维护的Git 存储库的详细信息。 清单文件默认命名为 west.yml； 这可以使用局部配置中的 manifest.file 配置选项来覆盖。
    您使用 west update 命令根据清单文件的内容更新工作区的Git 存储库。

项目
    项目是由 west 管理的 Git 存储库。 项目在清单文件中定义，可以位于工作区内的任何位置。 在上面的示例工作区中，zcbor 和 net-tools 是项目。
    默认情况下，Zephyr 构建系统使用 west 来获取工作区中所有项目的位置。

拓展命令
    west 已知的任何存储库（清单存储库或任何项目存储库）都可以定义针对该工作区的扩展命令。
    zephyr 存储库使用此功能来提供特定于 Zephyr 的命令，例如 west build。 将这些定义为扩展命令使 west 的核心与任何工作区的 Zephyr 版本等的细节无关。


ignored files
    工作区中可以包含不受 west 管理的文件、目录、Git 存储库。 west 基本上会忽略工作区中除 .west 目录、清单存储库和清单文件中指定的项目之外的任何内容。

west init 和 west update 命令
******************************

与工作区相关的两个最重要的命令是 west init 和 west update。

``west init``
----------------

这个命令创建一个 west 工作区。通常会像这样运行：

.. code-block:: bash

    west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.5.0 zephyrproject


这将会：

#. 创建名为 :file:`zephyrproject` 的顶级目录，以及其中的 :file:`.west` 和 :file:`.west/config` 目录。
#. 从 https://github.com/zephyrproject-rtos/zephyr 克隆清单存储库，将其放入 :file:`zephyrproject/zephyr`
#. 将克隆下来的本地的zephyr仓库切换至 v2.5.0 的 git tag。
#. 在 :file:`.west/config` 中将 ``manifest.path`` 设置为 ``zephyr``
#. 将 ``manifest.file`` 设置为 ``west.yml``

到目前为止工作区基本可以使用了； 只需要运行 west update 将其余项目克隆到工作区即可完成。

``west update``
----------------

此命令检查并确认清单文件中的项目是否在工作区中。

west update 命令通过以下方式读取清单文件的内容：

#. 寻找顶级目录。 在上面的 west init 示例中，这意味着找到 zephyrproject。
#. 在顶级目录中加载 .west/config 以读取 manifest.path（例如 zephyr）和 manifest.file（例如 west.yml）选项。
#. 加载通过上述信息确认出的清单文件（例如 zephyrproject/zephyr/west.yml）。

然后它使用清单文件来决定缺失的项目应该放在工作区中的什么地方，从哪些 URL 克隆它们，以及应该在本地签出哪些 Git 修订版。 对于已经存在的项目存储库，通过在清单文件中获取和检查它们各自的 Git 信息来更新。
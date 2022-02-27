安装 West
##########

West 使用 Python3 编写并用 [PyPi](https://pypi.org/project/west/) 分发。使用 pip3 安装或升级 West：

Linux：

.. code-block:: bash

    pip3 install --user -U west


Windows 和 macOS：

.. code-block:: bash

    pip3 install -U west

.. note::
    有关使用 --user 参数的更多说明，请参阅 Python 和 pip。

之后，您可以运行 pip3 show -f West 以获取有关安装 West 二进制文件和相关文件的位置的信息。

安装 west 后，您可以使用它来克隆 Zephyr 存储库。

结构
****

West 的代码通过 PyPI 在名为 West 的 Python 包中分发。 该发行版包括一个同样名为 west （Windows 上为 west.exe）的可执行文件。

安装 west 时，启动器由 pip3 放置在用户文件系统中的某个位置（确切位置取决于操作系统，但应该在 PATH 环境变量中）。 此启动器是运行两个内置命令（如 west init、west update）以及在工作区中发现的任何扩展命令的入口点。

除了它的命令行接口，您还可以直接使用 west 的 Python API。 有关详细信息，请参阅west API。

启用 shell 自动补全
*******************

West 目前支持以下场景的 shell 自动补全：

- Linux: bash
- macOS: bash
- Windows: 不支持

为了启用 shell 自动补全，您需要获取相应的自动补全脚本，并在每次进入新的 shell 会话时启用它。

要获取自动补全脚本，您可以使用 west completion 命令：

.. code-block:: bash

    cd /path/to/zephyr/
    # 进入zephyr所在目录
    west completion bash > ~/west-completion.bash


.. note::
    更新 Zephyr 时，请记住使用 west completion 更新补全脚本的本地副本。

接下来，您需要将 west-completion.bash 导入到您的 bash shell 中。

在 Linux 上，您有以下选项：

- 将 west-completion.bash 复制到 /etc/bash_completion.d/。
- 将 west-completion.bash 复制到 /usr/share/bash-completion/completions/。
- 将 west-completion.bash 复制到本地文件夹并从 ~/.bashrc 脚本中使用 source 命令启用它。

在 macOS 上，您有以下选项：

- 将 west-completion.bash 复制到本地文件夹并从 ~/.bash_profile 使用 source 命令启用它。
- 使用 brew 安装 bash-completion 包：

    .. code-block:: bash

        brew install bash-completion


    然后在你的 ~/.bash_profile 中source 命令启用补全脚本：

    .. code-block:: bash

        source /usr/local/etc/profile.d/bash_completion.sh


    最后将 west-completion.bash 复制到 /usr/local/etc/bash_completion.d/。
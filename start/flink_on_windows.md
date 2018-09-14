# 在 Windows 中运行 Flink

如果你想在本地 Windows 机器上运行 Flink，需要[下载](http://flink.apache.org/downloads.html)并解压 Flink 二进制包。完成之后你可以使用 **Windows 批处理** 文件（`.bat`）或 **Cygwin** 来运行 Flink 作业管理器。

## 使用 Windows 批处理文件启动

使用 **Windows 命令行**来启动 Flink，需要打开命令行窗口，切换至 Flink 的 `bin/` 目录，然后运行 `start-cluster.bat`。

注意：必须将你的 Java 运行环境的 ``bin`` 目录加入 Windows 的 ``%PATH%`` 变量中。你可以参照[此指南](http://www.java.com/en/download/help/path.xml)来将 Java 加入 ``%PATH%`` 变量。

```bash
$ cd flink
$ cd bin
$ start-cluster.bat
Starting a local cluster with one JobManager process and one TaskManager process.
You can terminate the processes via CTRL-C in the spawned shell windows.
Web interface by default on http://localhost:8081/.
```

在看到以上界面后，你需要再打开另一个终端窗口来使用 `flink.bat` 运行作业。

## 使用 Cygwin 与 Unix 脚本启动

启动 **Cygwin** 终端，进入 Flink 目录并运行 `start-cluster.sh` 脚本：

```bash
$ cd flink
$ bin/start-cluster.sh
Starting cluster.
```

## 从 Git 安装 Flink

如果你希望从 git repository 来安装 Flink，需要使用 Windows 下的 git shell，Cygwin 会提示类似如下的错误：

```bash
c:/flink/bin/start-cluster.sh: line 30: $'\r': command not found
```

由于 git 在 Windows 下运行时，会自动将 UNIX 换行符转换成 Windows 风格的换行符，而 Cygwin 只能处理 UNIX 风格的换行符，因此产生了这个错误。解决方案是将 Cygwin 设置按照以下步骤修改为正确的换行符格式：

1. 启动 Cygwin shell。

2. 输入以下命令确定你的主目录：

    ```bash
    cd; pwd
    ```

    它将会返回 Cygwin 的 root 路径。

3. 使用 NotePad、WordPad 或者别的文本编辑器打开主目录下的 `.bash_profile`，将以下几行加入到此文件的最后：（如果此文件不存在，则需要你自己创建它）

```bash
export SHELLOPTS
set -o igncr
```

保存文件并启动一个新的 bash shell。

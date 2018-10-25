---
title:  "在windows上运行Flink"
nav-parent_id: start
nav-pos: 12
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->
在windows上运行Flink
如果想在Windows机器本地运行Flink，则要[下载](http://flink.apache.org/downloads.html) 并把包进行解压缩。之后，可以使用 **Windows Batch** 文件 (`.bat`)， 或使用 **Cygwin** 来启动 Flink Jobmanager.

## 用 Windows Batch 启动

To start Flink in local mode from the用 *Windows Batch* 本地模式来启动 Flink，打开命令行，找到 Flink 的 `bin/` 文件夹并运行 `start-local.bat`.

注意：Windows 的 ``%PATH%`` 变量必须配置 Java Runtime Environment ``bin`` 。参考 [指南](http://www.java.com/en/download/help/path.xml) 来添加 Java ``%PATH%`` 变量。

~~~bash
$ cd flink
$ cd bin
$ start-local.bat
启动 Flink job manager. 默认 Web 地址为 http://localhost:8081/.
不要关闭该窗口，可通过 Ctrl+C 停止 job manager。
~~~

之后，可以再打开一个窗口用 `flink.bat` 跑任务。

{% top %}

## 使用 Cygwin 和 Unix 脚本启动任务

使用 *Cygwin* 需要启动 Cygwin 终端，找到 Flink 目录，启动 `start-local.sh` 脚本：

~~~bash
$ cd flink
$ bin/start-local.sh
启动 jobmanager.
~~~

{% top %}

## 从 Git 安装 Flink

如果您正在从 git repo 源中安装 Flink 并且您使用的是 Windows git shell，那么 Cygwin 可能会产生类似问题：

~~~bash
c:/flink/bin/start-local.sh: line 30: $'\r': command not found
~~~

该错误原因为在 Windows 中运行时，git 会自动将 UNIX 行结尾转换为Windows样式行结尾。 问题是 Cygwin 只能处理 UNIX 样式的行结尾。 解决方法可通过以下三个步骤调整 Cygwin 配置产生正确的行结尾：

1. 启动 Cygwin 脚本.

2. 查看主路径

    ~~~bash
    cd; pwd
    ~~~

    该命令会返回 Cygwin 根目录。

3. 使用 NotePad，WordPad 或其他编辑器打开主目录的 `.bash_profile` 文件，并附加一下内容：（如果该文件不存在，需自己创建）

~~~bash
export SHELLOPTS
set -o igncr
~~~

保存改文件并打开一个新的bash窗口。

{% top %}

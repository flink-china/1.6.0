# IDE 设置

本章将简介如何将 Flink 项目导入到 IDE 中以进行开发。为了编写 Flink 程序，请参见 [Java API]({{ site.baseurl }}/quickstart/java_api_quickstart.html) 和  [Scala API]({{ site.baseurl }}/quickstart/scala_api_quickstart.html) 等指南。

**注意：**如果你的 IDE 出现某些问题，可以先尝试运行 Maven 命令行（`mvn clean package -DskipTests`）以排除一些可能存在的 bug 并进行相关设置。

## 准备工作

在开始前，请先从我们的 [repositories](https://flink.apache.org/community.html#source-code) 下载 Flink 文件，如下所示：

```bash
git clone https://github.com/apache/flink.git
```

## IntelliJ IDEA

下面是为开发 Flink core 所设置 IntelliJ IDEA IDE 的一份简明指南。由于 Eclipse 在混合使用 Scale 和 Java 项目时会出现一些众所周知的问题，现在越来越多的贡献者都迁移到了 IntelliJ IDEA。

下面的文档描述了如何一步步为 Flink 设置 IntelliJ IDEA 2016.2.5 版本（[https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/)）。

### 安装 Scala 插件

IntelliJ 在安装时已经提供并配置好了 Scale 插件。如果没有安装此插件，请在导入 Flink 前按照以下操作来启用 IDE 对 Scala 项目与文件的支持：

1. 进入 IntelliJ 插件设置（IntelliJ IDEA -> Preferences -> Plugins）并点击“Install Jetbrains plugin...”。
2. 选择并安装“Scala”插件。
3. 重启 IntelliJ。

### 导入 Flink

1. 启动 IntelliJ IDEA，选择“Import Project”。
2. 选择 Flink repository 的根目录。
3. 选择“Import project from external model”，接着选择“Maven”。
4. 其余全部保留默认设置，点击“Next”直到出现选择 SDK 界面。
5. 如果这里还没有任何 SDK，则点击左上角的“+”自己创建一个。选择你的 JDK 根目录，点击“OK”。如果已经存在 SDK，则直接选择你的 SDK 即可。
6. 重复点击“Next”，直到完成导入。
7. 在导入好的 Flink 项目，点击 project -> Maven -> Generate Sources and Update Folders。请注意这将在你的本地 Maven 仓库中安装 Flink 库。（如 `/home/*-your-user-*/.m2/repository/org/apache/flink/`）。或者使用 `mvn clean package -DskipTests` 安装 IDE 正常工作所需要的文件。
8. 构建 Project（点击 Build -> Make Project）

### 配置 Java 的 Checkstyle
在 IntelliJ 中，可以使用 Checkstyle-IDEA 插件来支持 checkstyle。

1. 从 IntelliJ 插件仓库中安装“Checkstyle-IDEA”插件。
2. 点击 Settings -> Other Settings -> Checkstyle 来配置 Checkstyle-IDEA 插件。
3. 将“Scan Scope”设为“Only Java sources (including tests)”。
4. 在“Checkstyle Version”下拉菜单中选择 _8.4_，点击 apply。**本步骤很重要，请勿跳过**
5. 在“Configuration File“面板中，点击加号图标来添加一个新的配置：
    1. 将“Description”设为“Flink”。
    2. 选择“Use a local Checkstyle file”，然后将它设置为你的 repository 中的  `"tools/maven/checkstyle.xml"`。
    3. 勾选“Store relative to project location”，点击“Next”。
    4. 配置“checkstyle.suppressions.file”属性为 `"suppressions.xml"`，然后依次点击“Next”和“Finish”。
6. 将“Flink”设置为唯一一个激活的配置，然后依次点击“Apply”和“OK”。
7. Checkstyle 将会在发现任何违背 Checkstyle 的情况时在编辑器内告警。

在装好插件之后，你就可以直接通过点击 Settings -> Editor -> Code Style -> Java -> ‘靠近 Scheme dropbox 的齿轮图标’来导入 `"tools/maven/checkstyle.xml"`。这将自动改变导入模块的格式。

你可以通过打开 Checkstyle 窗口，点击“Check Module”按钮来扫描整个模块。正常情况下，扫描应该不出现错误。

**注意：** `flink-core`、`flink-optimizer` 和 `flink-runtime` 等模块不会被 checkstyle 完全覆盖到。不过还是请您确保你在这些模块中添加或修改的代码符合 checkstyle 规则。

### 配置 Scala 的 Checkstyle

选择 Settings -> Editor -> Inspections，接着搜索“Scala style inspections”，选中它以启用 Intellij 的 scalastyle。你也可以将 `"tools/maven/scalastyle_config.xml"` 放在 `"<root>/.idea"` 或 `"<root>/project"` 目录下以实现此功能。

## Eclipse

**注意：**依据我们的经验，本设置可能无法适用与 Flink，因为 Scala IDE 3.0.3 依赖于老版的 Eclipse，且 Scala IDE 4.4.1 使用了不兼容的 Scala 版本。

**我们更推荐使用 IntelliJ 来代替 Eclipse（请参见[前文](#intellij-idea)）

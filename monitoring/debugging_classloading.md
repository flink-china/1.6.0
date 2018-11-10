# Flink类加载概述
Flink应用程序在运行过程中，随着时间的推移，会加载各种各样的类。 这些类可以分为两种：
* Java Classpath：包括JDK的库和Flink lib目录下的所有类（Flink的类和一些核心依赖）。
* 动态用户代码：通过Web/Rest/命令行上传的Job Jar包中的类。每个Job的类会被动态的加载和卸载。

*译着注：Flink Job的类会随着Job的Submit/Cannel而动态的加载/卸载。Flink自建线程上下文类加载器并重写了loadClass()方法，会首先查找存放Job  Jar包的目录，如果不存在，会依据双亲委派模型的规则委派给应用程序类加载器。不同的Job运行在不同的线程，并且存放Jar包的目录也会不一样。*

类具体属于哪一种，和Flink的部署模式息息相关。大体来讲，如果先启动Flink进程，后提交Job，Job中的类动态加载。如果Flink进程随Job/Application一起启动（例如Docker的Job模式），或者Appliation把Flink的组件拉起来（例如YARN的Flink Job模式），所有的类都包含着在Classpath中。

不同部署模式下的更多细节：

**Standalone Session模式**

Standalone Session模式启动JobManager和TaskManager时, ClassPath会指向Flink的框架类。该模式用户需要通过Rest/命令行向Session提交Job，Job中的类动态加载。

**Docker / Kubernetes Session模式**

和Standalon Session模式类似，Docker/ Kubernets Session模式也是先启动一个包含JobManager和Taskmanager的集群。用户通过Rest/命令行的形式向Session提交Job。Java Classpath中包含Flink的框架类，Job中的用户类在提交后动态加载。


**YARN 模式**

YARN可以细分成两种部署模式：
1. 直接向YARN提交Flink Job/Application(运行bin/flink run -m yarn-cluster ...)。YARN会为该Job启动TaskManagers和JobManagers，对应JVM的Classpath中既包含了Flink的框架类，也包含Job的用户代码类。该场景下，不涉及类的动态加载。
2. YARN Sesion模式。该模式先启动JobManagers和TaskManagers，Flink的框架类都在Classpath中，Job中的用户类动态加载。

**Mesos 模式**

按照[文档](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/mesos.html)部署的Mesos模式和YARN Session模式类似：Flink框架类包含在JobManager/TaskManager的Classpath中，Job中的用户类在提交后动态加载。


# 反向类加载和类加载器的解析顺序
动态类加载场景下，有一个典型的两层类加载器结构：（1）Java的应用程序类加载器(Application Classloader)，用于加载Classpath的所有类；（2）动态的自定义类加载器(User classLoader)，负责加载用户类Jar包。应用程序类加载器是自定义类加载器的父加载器。

默认情况下，Flink反转类的加载顺序，即先通过自定义的类加载器加载类，只有当类不属于动态加载的用户类时，才会用父加载器（Application Classloader）进行加载。

反向类加载的好处是，不同的Job可以采用不同版本的Flink 核心类，以解决版本不兼容问题。这种加载机制可以避免如IllegalAccessError、NoSuchMethodError等常见的依赖冲突异常。不同的用户代码直接用不同的类副本（Flink核心类或某些依赖类可以和用户代码用不同的副本）。能搞定大多数的情况，且无需额外的Job配置。

但是反向类加载也会导致一些问题，例如下面会描述的"X cannot be cast to X"问题。可以将类加载器的解析顺序改为Java的默认加载模式。具体方式：设置Flink的配置项"classloader.resolve-order"值为"parent-first"（默认为“child-first”）。

需要注意的是，即使在child-fitst加载模式下，有些类也必须采用parent-first的加载顺序。因为有些类可能是Flink 和用户代码共享的类，或者面向用户代码的API。实现方式是将这个类所在的包加入到配置项classloader.parent-first-patterns-default和classloader.parent-first-patterns-additional中。新加parent-fisrt包时，请通过classloader.parent-first-patterns-additional设置。

# 避免动态类加载
所有组件，包括JobManager、TaskManager、Clinet、ApplicationMaster等，启动时都会将Classpath写入日志，可以在日志文件头部的环境信息中发现相关描述。

如果JobManager和TaskNanager是独立于Job启动的，可以将Job的Jar包放置在Flink的lib目录，以避开动态加载。

将Job的JAR包放置在Flin的Lib目录后，JAR包既可以在Classpath中被找到（即被应用程序类加载器找到），也可以被自定义的类加载器找到。因为应用程序类加载器是自定义类加载器的父加载器，且Java采用双亲委派模型，所以该Jar包中的类都只会被加载一次。

对于不能将整个Job Jar包放在Flink lib包的场景，例如Session方式的部署模式中，Session被多个Job共用。可以将一些公共的库放置在Flink的Lib目录。


# Job中手动加载类
某些场景，Transformation、Source、Sink中需要手动加载类（通过反射动态加载类）。此时需要拿到能访问Job类的类加载器。Function可以先继承RichFunction（例如RichMapFunction、RichWindowFunction），然后通过getRuntimeContext().getUserCodeClassLoader()获取用户类加载器，Source和Sink类似。


# X cannot be cast to X 异常
采取有动态类加载的部署方式时，可能会出现“com.foo.X cannot be cast to com.foo.X”的异常。意味着不同版本的com.foo.X被不同的类加器加载了。其中某个类被赋值给另外一个类。

通常的原因是Lib不适用Flink的反向类加载方式。可以通过Flink配置项“classloader.resolve-order: parent-first”来关闭反向类加载。或通过 配置项“classloader.parent-first-patterns-additional”将对应库从反向加载类中排除。

另一种可能是因为Java实例的缓存，例如因为Apache Avro或者Guava的缓存对象驻留。解决办法是采取不会动态类加载的部署方式，或者保证对应的库是动态加载代码的一部分，此时，库不能放在Flink的lib目录下，打包时也需要带上依赖包打成fat-jar/uber-jar。

# 卸载动态加载的类
所有类可以动态加载的场景，都依赖类可以被卸载。类卸载意味着垃圾回收器发现没有类没有对象了，然后移除类，包括代码、静态变量、元数据等。


当Taskmanager启动（或重启）Task时，会加载Task的代码。如果类不能卸载，就会造成内存泄漏。随着新版本类的加载，类的数量会随时间越积越多，典型的表现是产生**OutOfMemoryError: Metaspace**的异常。

类泄漏的常见原因和解决办法：
* *线程残留：* 确保应用停止时关闭了所有的线程，残留的线程会消耗资源，并且保留了对象的引用，会阻止垃圾回收器卸载类。
* *驻留：* 避免将对象缓存在超出Functions/Sources/Sinks生命周期的结构中。例如Guava的驻留，或Avro在序列化器中类/对象的缓存。


# 通过maven-shade-plugin解决和Flink的依赖冲突
一种在应用开发者侧避开依赖冲突的办法是，通过将类隐去来避免依赖传递。

Apache Maven提供了"maven-shade-plugin"插件，可以在代码编译后更换类所在的包，用户代码不会受影响。例如，在用户代码中引用了AWS SDK中的包“com.amazonaws packages”。shade插件可以通过改变字节码的方式将包的引用改为"org.myorg.shaded.com.amazonaws"，以达到调用自定义AWS SDK的效果。

此文解释了[通过shade插件重定向类](https://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html)。

注意，大部分Flink的依赖包，例如guava, netty, jackson等都已经被Flink的维护人员隐去，用户不用太担心。

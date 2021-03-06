- [JVM性能调优](#JVM性能调优)
  - [基础故障处理工具](#基础故障处理工具)
    - [jps-虚拟机进程状况工具](#jps-虚拟机进程状况工具)
    - [jstat-虚拟机统计信息监视工具](#jstat-虚拟机统计信息监视工具)
    - [jinfo-Java配置信息工具](#jinfo-Java配置信息工具)
    - [jmap-Java内存映像工具](#jmap-Java内存映像工具)
    - [jhat-虚拟机堆转储快照分析工具](#jhat-虚拟机堆转储快照分析工具)
    - [jstack-Java堆栈跟踪工具](#jstack-Java堆栈跟踪工具)
  - [进阶故障处理工具](#进阶故障处理工具)
    - [HSDB-基于服务性代理的调试工具](#HSDB-基于服务性代理的调试工具)
    - [JConsole-Java监视与管理控制台](#JConsole-Java监视与管理控制台)
    - [VisualVM-多合一故障处理工具](#VisualVM-多合一故障处理工具)
    - [Arthas-Alibaba开源的Java诊断工具](#Arthas-Alibaba开源的Java诊断工具)
  - [性能调优记录](#性能调优记录)
    - [程序死循环问题排查](#程序死循环问题排查)
    - [程序死锁问题排查](#程序死锁问题排查)
    - [生成与分析Dump文件](#生成与分析Dump文件)
    - [OOM异常问题排查](#OOM异常问题排查)
    - [线上业务日志切面](#线上业务日志切面)
    - [CPU占用过高问题排查](#CPU占用过高问题排查)
    - [亿级流量系统调优案例](#亿级流量系统调优案例)

# JVM性能调优

> 文章标题中包含了“性能调优”，但是，我个人觉得，调优案例这一部分，目前手上有的书本资料远远不够，《深入理解JAVA虚拟机》只是描述了一下案例简介，完全没有深入分析（当然也理解，篇幅有限，不可能做深入分析）。这一部分，需要结合实际项目来分析，成本比较高，我手上一时半会没有好的项目例子。因此呢，我将文章核心放小一些：至少对JVM各种运维工具进行一次介绍，介绍它们的基本功能，能实战演练的就尝试一次，会涉及到的工具有：jps、jstat、jinfo、jmap、jhat、jstack、JHSDB、JConsole、VisualVM、Arthas。其中重点介绍 VisualVM 和 Arthas 两款工具

> 在不断学习的过程中，我接触到了许多实战例子，我将它们逐一记录了下来，同时，我想到是否可以多维度的模拟性能调优案例来自行测试，比如搭建一套Spring Web应用，不断模拟用户端请求，创建大量的Bean，通过VisualVM或者Arthas工具查看内存调用情况。至此我的思路就展开了

### 基础故障处理工具

安装了JAVA后，在 %JAVA_HOME%/bin/ 下存在许多工具，它们是JDK内部自带的工具，用于问题排查和分析

这些工具的源码都在 https://github.com/peteryuanpan/openjdk-8u40-source-code-mirror/tree/master/jdk/src/share/classes/sun/tools ，都是开源的，且是JAVA代码

![image](https://user-images.githubusercontent.com/10209135/100352674-505c8d80-3028-11eb-809e-90fce04409af.png)

#### jps-虚拟机进程状况工具

jps，JVM Process Status Tool，显示指定系统内所有的 Hotspot 虚拟机进程

命令指南
```cmd
jps -help
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]
```

通过如下方法可以打印本地电脑上 jps 命令输出的本地 hsperfdata 文件所在文件夹

```java
package jtools;

import sun.misc.VMSupport;

public class JPSTempFile {

    public static void main(String[] args) {
        System.out.println(VMSupport.getVMTemporaryDirectory());
    }
}
```

输出结果
```
C:\Users\Admin\AppData\Local\Temp\
```

最终路径 = 输出结果 + "hsperfdata_<Username>"（我这里是Admin）

![image](https://user-images.githubusercontent.com/10209135/100350483-ef7f8600-3024-11eb-96ee-d0e93d7191cc.png)

再查询jps命令，可以看出来是一一对应的

```cmd
jps -l
15976 sun.tools.jps.Jps
10876
5692 org.jetbrains.jps.cmdline.Launcher
```

#### jstat-虚拟机统计信息监视工具

jstat，JVM Statistics Monitoring Tool，是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中 类加载、内存、垃圾收集、JIT编译 等运行数据。jstat 可以理解为 jconsole、jvisualvm 图形界面工具的命令行版本

命令指南

```cmd
jstat
invalid argument count
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
```

|命令|解释|
|--|--|
|-class pid|显示加载class的数量，及所占空间等信息|
|-compiler pid|显示VM实时编译的数量等信息|
|-gc pid|可以显示gc的信息，查看gc的次数，及时间|
|-gccapacity pid|可以显示VM内存中三代（young,old,perm）对象的使用和占用大小|
|-gcnew pid|new对象的信息|
|-gcnewcapacity pid|new对象的信息及其占用量|
|-gcold pid|old对象的信息|
|-gcoldcapacity pid|old对象的信息及其占用量|
|-gcpermcapacity pid| perm对象的信息及其占用量|
|-util pid|统计gc信息统计|
|-printcompilation pid|当前VM执行的信息|

以 -gc 为例子，先通过 jps 获取 pid，然后执行 jstat -gc < pid >，得到如下结果

```cmd
jstat -gc 16952
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
2048.0 2048.0  0.0    0.0   12800.0   5329.2   34304.0      0.0     4480.0 776.7  384.0   76.6       0    0.000   0      0.000    0.000
```

#### jinfo-Java配置信息工具

jinfo，Configuration Info for Java，作用是实时查看和调整虚拟机各项参数

命令指南

```cmd
jinfo
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
```

先通过 jps 获取 pid，执行 jinfo < pid >，得到如下结果

```cmd
jinfo 16952
Attaching to process ID 16952, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.231-b11
Java System Properties:

java.runtime.name = Java(TM) SE Runtime Environment
java.vm.version = 25.231-b11
sun.boot.library.path = C:\Program Files\Java\jdk1.8.0_231\jre\bin
java.vendor.url = http://java.oracle.com/
java.vm.vendor = Oracle Corporation
path.separator = ;
file.encoding.pkg = sun.io
java.vm.name = Java HotSpot(TM) 64-Bit Server VM
sun.os.patch.level =
sun.java.launcher = SUN_STANDARD
user.script =
user.country = CN
user.dir = D:\workspace\luban-jvm-research
java.vm.specification.name = Java Virtual Machine Specification
java.runtime.version = 1.8.0_231-b11
java.awt.graphicsenv = sun.awt.Win32GraphicsEnvironment
os.arch = amd64
java.endorsed.dirs = C:\Program Files\Java\jdk1.8.0_231\jre\lib\endorsed
line.separator =

java.io.tmpdir = C:\Users\Admin\AppData\Local\Temp\
java.vm.specification.vendor = Oracle Corporation
user.variant =
os.name = Windows 10
sun.jnu.encoding = GBK
java.library.path = C:\Program Files\Java\jdk1.8.0_231\bin;C:\Windows\Sun\Java\bin;C:\Windows\system32;C:\Windows;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;D:\download\qshell-windows-x64-v2.4.1.exe;C:\Program Files\NVIDIA Corporation\NVIDIA NvDLISR;D:\qiniu\tools;D:\download\arthas-packaging-3.4.4-bin\;D:\ffmpeg\bin;D:\Git\cmd;C:\MinGW\bin;C:\Program Files\Java\jdk1.8.0_231\bin;D:\GnuWin32\bin;C:\Users\Admin\Desktop\plan\2020\gradle-5.5.1-bin\gradle-5.5.1\bin;C:\Program Files\MySQL\MySQL Server 8.0\bin;D:\download\v2ray-windows-64;D:\erl-23.1\bin;D:\RabbitMQ Server\rabbitmq_server-3.8.9\sbin;D:\download\Redis-x64-5.0.9;D:\nodejs\;D:\python2.7.18;C:\ProgramData\chocolatey\bin;C:\Program Files\dotnet\;C:\Users\Admin\AppData\Local\Microsoft\WindowsApps;;D:\Microsoft VS Code\bin;C:\Users\Admin\AppData\Roaming\npm;C:\Users\Admin\.dotnet\tools;.
java.specification.name = Java Platform API Specification
java.class.version = 52.0
sun.management.compiler = HotSpot 64-Bit Tiered Compilers
os.version = 10.0
user.home = C:\Users\Admin
user.timezone = Asia/Shanghai
java.awt.printerjob = sun.awt.windows.WPrinterJob
file.encoding = UTF-8
java.specification.version = 1.8
user.name = Admin
java.class.path = C:\Program Files\Java\jdk1.8.0_231\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\deploy.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\access-bridge-64.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\cldrdata.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\dnsns.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\jaccess.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\jfxrt.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\localedata.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\nashorn.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunec.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunjce_provider.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunmscapi.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunpkcs11.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\zipfs.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\javaws.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfxswt.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\management-agent.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\plugin.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\rt.jar;D:\workspace\luban-jvm-research\target\classes;C:\Users\Admin\.m2\repository\org\openjdk\jol\jol-core\0.10\jol-core-0.10.jar;C:\Users\Admin\.m2\repository\mysql\mysql-connector-java\8.0.20\mysql-connector-java-8.0.20.jar;C:\Users\Admin\.m2\repository\com\google\protobuf\protobuf-java\3.6.1\protobuf-java-3.6.1.jar;C:\Users\Admin\.m2\repository\cglib\cglib\2.2.2\cglib-2.2.2.jar;C:\Users\Admin\.m2\repository\asm\asm\3.3.1\asm-3.3.1.jar;D:\IntelliJ IDEA 2020.2.3\lib\idea_rt.jar
java.vm.specification.version = 1.8
sun.arch.data.model = 64
sun.java.command = com.peter.jvm.example2.oom.HeapOOM
java.home = C:\Program Files\Java\jdk1.8.0_231\jre
user.language = zh
java.specification.vendor = Oracle Corporation
awt.toolkit = sun.awt.windows.WToolkit
java.vm.info = mixed mode
java.version = 1.8.0_231
java.ext.dirs = C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext;C:\Windows\Sun\Java\lib\ext
sun.boot.class.path = C:\Program Files\Java\jdk1.8.0_231\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\rt.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\sunrsasign.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_231\jre\classes
java.vendor = Oracle Corporation
file.separator = \
java.vendor.url.bug = http://bugreport.sun.com/bugreport/
sun.io.unicode.encoding = UnicodeLittle
sun.cpu.endian = little
sun.desktop = windows
sun.cpu.isalist = amd64

VM Flags:
Non-default VM flags: -XX:CICompilerCount=3 -XX:+HeapDumpOnOutOfMemoryError -XX:InitialHeapSize=52428800 -XX:MaxHeapSize=52428800 -XX:MaxNewSize=17301504 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=17301504 -XX:OldSize=35127296 -XX:+PrintGC -XX:+PrintGCDetails -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
Command line:  -Xms50M -Xmx50M -XX:+HeapDumpOnOutOfMemoryError -verbose:gc -XX:+PrintGCDetails -javaagent:D:\IntelliJ IDEA 2020.2.3\lib\idea_rt.jar=56037:D:\IntelliJ IDEA 2020.2.3\bin -Dfile.encoding=UTF-8
```

可以通过 jinfo -flag +-name | name=value pid 来修改参数值

可以通过 jinfo -flag name pid 来查看参数值，得到如下结果

```
jinfo -flag MaxNewSize 16952
-XX:MaxNewSize=17301504
```

同样的，还有其他办法也能达到 jinfo 的效果，比如 jps -v，得到如下结果

```cmd
jps -v
10536 Launcher -Xmx700m -Djava.awt.headless=true -Djava.endorsed.dirs="" -Djdt.compiler.useSingleThread=true -Dpreload.project.path=D:/workspace/luban-jvm-research -Dpreload.config.path=C:/Users/Admin/AppData/Roaming/JetBrains/IntelliJIdea2020.2/options -Dcompile.parallel=false -Drebuild.on.dependency.change=true -Dio.netty.initialSeedUniquifier=-4213499304903621881 -Dfile.encoding=GBK -Duser.language=zh -Duser.country=CN -Didea.paths.selector=IntelliJIdea2020.2 -Didea.home.path=D:\IntelliJ IDEA 2020.2.3 -Didea.config.path=C:\Users\Admin\AppData\Roaming\JetBrains\IntelliJIdea2020.2 -Didea.plugins.path=C:\Users\Admin\AppData\Roaming\JetBrains\IntelliJIdea2020.2\plugins -Djps.log.dir=C:/Users/Admin/AppData/Local/JetBrains/IntelliJIdea2020.2/log/build-log -Djps.fallback.jdk.home=D:/IntelliJ IDEA 2020.2.3/jbr -Djps.fallback.jdk.version=11.0.8 -Dio.netty.noUnsafe=true -Djava.io.tmpdir=C:/Users/Admin/AppData/Local/JetBrains/IntelliJIdea2020.2/compile-server/luban-jvm-research_3f84df24/_temp_ -Djps.backward.ref.index.builder=true
16952 HeapOOM -Xms50M -Xmx50M -XX:+HeapDumpOnOutOfMemoryError -verbose:gc -XX:+PrintGCDetails -javaagent:D:\IntelliJ IDEA 2020.2.3\lib\idea_rt.jar=56037:D:\IntelliJ IDEA 2020.2.3\bin -Dfile.encoding=UTF-8
2008 Jps -Dapplication.home=C:\Program Files\Java\jdk1.8.0_231 -Xms8m
10876  exit -Xms128m -Xmx2039m -XX:ReservedCodeCacheSize=240m -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -XX:CICompilerCount=2 -Dsun.io.useCanonPrefixCache=false -Djdk.http.auth.tunneling.disabledSchemes="" -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Djdk.attach.allowAttachSelf=true -Dkotlinx.coroutines.debug=off -Djdk.module.illegalAccess.silent=true -javaagent:C:\Users\Public\.jetbrains\jetbrains-agent-v3.2.0.0f1f.69e=6e68f9eb,LFq51qqupnaiTNn39w6zATiOTxZI2JYuRJEBlzmUDv4zeeNlXhMgJZVb0q5QkLr+CIUrSuNB7ucifrGXawLB4qswPOXYG7+ItDNUR/9UkLTUWlnHLX07hnR1USOrWIjTmbytcIKEdaI6x0RskyotuItj84xxoSBP/iRBW2EHpOc -Djb.vmOptionsFile=C:\Users\Admin\AppData\Roaming\JetBrains\IntelliJIdea2020.2\idea64.exe.vmoptions -Djava.library.path=D:\IntelliJ IDEA 2020.2.3\jbr\\bin;D:\IntelliJ IDEA 2020.2.3\jbr\\bin\server -Didea.jre.check=true -Dide.native.launcher=true -Didea.vendor.name=JetBrains -Didea.paths.selector=IntelliJIdea2020.2 -XX:ErrorFile=C:\Users\Admin\java_error_in_idea_%p.log -XX:HeapDumpPath=C:
```

java -XX:+PrintCommandLineFlags -version，查看JVM常用参数，得到如下结果

```cmd
-XX:InitialHeapSize=267373952 -XX:MaxHeapSize=4277983232 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)
```

java -XX:+PrintFlagsFinal -version，查看JVM主要参数，得到如下结果

```cmd
    uintx MarkStackSize                             = 4194304                             {product}
    uintx MarkStackSizeMax                          = 536870912                           {product}
    uintx MarkSweepAlwaysCompactCount               = 4                                   {product}
    uintx MarkSweepDeadRatio                        = 1                                   {product}
     intx MaxBCEAEstimateLevel                      = 5                                   {product}
     intx MaxBCEAEstimateSize                       = 150                                 {product}
    uintx MaxDirectMemorySize                       = 0                                   {product}
     bool MaxFDLimit                                = true                                {product}
    uintx MaxGCMinorPauseMillis                     = 4294967295                          {product}
    uintx MaxGCPauseMillis                          = 4294967295                          {product}
    uintx MaxHeapFreeRatio                          = 100                                 {manageable}
    uintx MaxHeapSize                              := 4278190080                          {product}
     intx MaxInlineLevel                            = 9                                   {product}
     intx MaxInlineSize                             = 35                                  {product}
     intx MaxJNILocalCapacity                       = 65536                               {product}
     intx MaxJavaStackTraceDepth                    = 1024                                {product}
     intx MaxJumpTableSize                          = 65000                               {C2 product}
     intx MaxJumpTableSparseness                    = 5                                   {C2 product}
     intx MaxLabelRootDepth                         = 1100                                {C2 product}
     intx MaxLoopPad                                = 11                                  {C2 product}
    uintx MaxMetaspaceExpansion                     = 5451776                             {product}
    uintx MaxMetaspaceFreeRatio                     = 70                                  {product}
    uintx MaxMetaspaceSize                          = 4294901760                          {product}
    uintx MaxNewSize                               := 1426063360                          {product}
     intx MaxNodeLimit                              = 75000                               {C2 product}

...

java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)
```

#### jmap-Java内存映像工具

jmap，Memory Map for Java，用于生成堆转储快照（一般称为 heapdump 或 dump 文件）

还有其他办法也能生成 dump 文件
- 通过参数 -XX:+HeapDumpOnOutOfMemoryError，让虚拟机在OOM异常出现之后自动生成 dump 文件，通过参数 -XX:HeapDumpPath= 可以指定 dump 文件生成目录
- 通过参数 -XX:+HeapDumpOnCtrlBeak，可以使用 ctrl + break 键让虚拟机生成 dump 文件
- Linux系统下，通过 kill -3 命令发送进程退出信号“吓唬”一下虚拟机，也能拿到 dump 文件

命令指南

```cmd
jmap
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -clstats             to print class loader statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
```

通过 jps 拿到 pid，执行 jmap -dump:format=b,file=1.bin < pid >，得到如下结果

```cmd
jmap -dump:format=b,file=1.bin 16868
Dumping heap to C:\Users\Admin\1.bin ...
Heap dump file created
```

会在执行目录下生成一份 1.bin 二进制文件，该文件可以从 jvisualvm 工具中导入进行分析，还可以通过寒泉子 https://memory.console.perfma.com/ 上传 dump 文件来进行分析

值得一提的是，jmap 有不少功能在 Windows 平台下是受限的，能使用 -dump、-histo 参数，而 -finalizerinfo、-heap、-permstat、-F 等参数都只能在 Linux / Solairs 平台下使用

#### jhat-虚拟机堆转储快照分析工具

jhat，JVM Heap Analysis Tool，与 jmap 搭配使用，来分析 jmap 生成的堆转储快照。jhat 内置了一个微型的 HTTP/HTML 服务器，生成 dump 文件的分析结果后，可以在浏览器中查看

但这个工具基本已经是“鸡肋”了，原因有二，一是一般不会在生产服务器上进行分析，而是导出到其他机器上，那么就没必要一定使用 jhat 了，二是 jhat 分析功能比较简陋，后面介绍的 VisualVM 会比它功能强大许多

通过 jmap 生成 1.bin 这样的 dump 文件，执行 jhat 1.bin，得到如下结果

```cmd
jhat 1.bin
Reading from 1.bin...
Dump file created Thu Nov 26 21:57:13 CST 2020
Snapshot read, resolving...
Resolving 87056 objects...
Chasing references, expect 17 dots.................
Eliminating duplicate references.................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

浏览器中访问 http://localhost:7000/ ，得到如下结果

（看到这样的HTML界面，还支持点击跳转，有没有一种很原始的感觉? 这应该是早期，或许是20世纪90年代，JAVA程序员的一种原始的调试方法）

![image](https://user-images.githubusercontent.com/10209135/100362563-8bfe5400-3036-11eb-81a0-b47a6300a4ab.png)

#### jstack-Java堆栈跟踪工具

jstack，Stack Trace for Java，用于生成虚拟机当前时刻的线程快照（一般称为 threaddump 或者 javacore 文件）

线程快照就是当前虚拟机每一条线程正在执行的方法堆栈的集合，生成快照的主要目的是定位线程出现长时间停顿的原因，如线程死锁、死循环、请求外部资源导致的长时间等待等

命令指南

```cmd
jstack
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
```

通过 jps 获取到 pid，执行 jstack < pid >

```cmd
jstack 16044
2020-11-26 22:43:05
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.231-b11 mixed mode):

"Service Thread" #10 daemon prio=9 os_prio=0 tid=0x00000000156ad800 nid=0x41ac runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x0000000015612800 nid=0x3650 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x0000000015610800 nid=0x4580 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x0000000015610000 nid=0x46d0 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x0000000015609000 nid=0x4588 runnable [0x0000000015c3e000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:171)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
        at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
        at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
        at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
        - locked <0x00000000ff112c00> (a java.io.InputStreamReader)
        at java.io.InputStreamReader.read(InputStreamReader.java:184)
        at java.io.BufferedReader.fill(BufferedReader.java:161)
        at java.io.BufferedReader.readLine(BufferedReader.java:324)
        - locked <0x00000000ff112c00> (a java.io.InputStreamReader)
        at java.io.BufferedReader.readLine(BufferedReader.java:389)
        at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:61)

"Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x0000000013bc2000 nid=0x4190 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x0000000015593000 nid=0x3dd4 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=1 tid=0x000000000367d800 nid=0x3ca8 in Object.wait() [0x000000001553f000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000fef88ed8> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
        - locked <0x00000000fef88ed8> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:216)

"Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x0000000013b9c000 nid=0x4424 in Object.wait() [0x000000001543f000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000fef86c00> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:502)
        at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
        - locked <0x00000000fef86c00> (a java.lang.ref.Reference$Lock)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"main" #1 prio=5 os_prio=0 tid=0x0000000003583800 nid=0x31f4 waiting on condition [0x000000000338f000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at com.peter.jvm.example2.oom.HeapOOM.main(HeapOOM.java:12)

"VM Thread" os_prio=2 tid=0x0000000013b78000 nid=0x19a0 runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000003599000 nid=0x4354 runnable

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x000000000359a800 nid=0x45dc runnable

"GC task thread#2 (ParallelGC)" os_prio=0 tid=0x000000000359c000 nid=0x3448 runnable

"GC task thread#3 (ParallelGC)" os_prio=0 tid=0x000000000359e800 nid=0x46dc runnable

"GC task thread#4 (ParallelGC)" os_prio=0 tid=0x00000000035a0800 nid=0x4610 runnable

"GC task thread#5 (ParallelGC)" os_prio=0 tid=0x00000000035a2000 nid=0x1180 runnable

"VM Periodic Task Thread" os_prio=2 tid=0x00000000156e9000 nid=0x3794 waiting on condition

JNI global references: 12
```

同样的，还可以通过 Thread.getAllStackTraces 来打印出如 jstack 一样的堆栈输出信息（Returns a map of stack traces for all live threads.）

```java
package jtools;

import java.util.Map;

public class JStackTest {

    public static void main(String[] args) {
        for (Map.Entry<Thread, StackTraceElement[]> stackTrace : Thread.getAllStackTraces().entrySet()) {
            Thread thread = stackTrace.getKey();
            System.out.println("\nThread：" + thread.getName() + "\n");
            StackTraceElement[] stack = stackTrace.getValue();
            for (StackTraceElement element : stack) {
                System.out.println("\t" + element + "\t");
            }
        }
    }
}
```

输出结果
```
Thread：main

	java.lang.Thread.dumpThreads(Native Method)	
	java.lang.Thread.getAllStackTraces(Thread.java:1610)	
	jtools.JStackTest.main(JStackTest.java:8)
  
Thread：Reference Handler

	java.lang.Object.wait(Native Method)	
	java.lang.Object.wait(Object.java:502)	
	java.lang.ref.Reference.tryHandlePending(Reference.java:191)	
	java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)	
  
Thread：Finalizer

	java.lang.Object.wait(Native Method)	
	java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)	
	java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)	
	java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:216)	

Thread：Monitor Ctrl-Break

	java.net.Socket.setImpl(Socket.java:520)	
	java.net.Socket.<init>(Socket.java:441)	
	java.net.Socket.<init>(Socket.java:228)	
	com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:56)	
```

### 进阶故障处理工具

上面介绍完了基于命令行的基础故障处理工具，下面是更高级的故障处理工具，除了 Arthas 外，JHSDB、JConsole、VisualVM 都具有图形界面功能

善于总结的朋友可以发现，这些更高级功能的工具，往往继承了前面多个基础故障处理工具的功能，比如 VisualVM 是同时具有 jps、jinfo、jmap、jhat、jstack 功能的

#### HSDB-基于服务性代理的调试工具

HSDB是一款基于服务性代理（Serviceability Agent，SA）实现的进程外调试工具。服务性代理是HotSpot虚拟机中一组用于映射Java虚拟机运行信息的、主要基于Java语言（含少量JNI代码）实现的API集合

HSDB能查看很多内存相关的数据信息，其中很多信息（比如klass、ArrayKlass）都是JVM源码中定义的数据结构

启动HSDB，打开cmd，cd 到 %JAVA_HOME$/lib 下，运行 java -cp sa-jdi.jar sun.jvm.hotspot.HSDB，会打开一个看似空白的界面

测试一段代码（让代码进入while循环）

```java
package com.luban.ziya.peter;

public class TEST1 {
    public static void main(String[] args) {
        while(true);
    }
}
```

通过 jps 查看进程ID（这里是5996），在 HSDB 界面左上角点击File > Attach to Hotspot process，输入5996，点击OK或回车

![image](https://user-images.githubusercontent.com/10209135/90361871-d6fce880-e091-11ea-85f8-6a3890caa34c.png)

这样就 attch 进入了一个进程，得到如下结果

![image](https://user-images.githubusercontent.com/10209135/90361965-09a6e100-e092-11ea-8145-a3fbe7548039.png)

下面贴一下不同的查询结果结果，具体查询过程就忽略了，可以自行研究或者查看文档

![image](https://user-images.githubusercontent.com/10209135/90362306-e466a280-e092-11ea-839c-0f6d1edaaa5a.png)

![image](https://user-images.githubusercontent.com/10209135/90372437-00724000-e0a3-11ea-814c-d2a4edce114c.png)

#### JConsole-Java监视与管理控制台

JConsole，Java Monitoring and Management Console，是一款基于JMX（Java Manage-mentExtensions）的可视化监视、管理工具。它的主要功能是通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整

Jconsole的功能性并不如（甚至远远不如）VisualVM，且界面比较陈旧、面板化，因此在这里并不打算对它进行过多介绍，只是提一下有这么一个工具

在命令行中执行 jconsole，即可打开这个图形界面工具，其中之一界面如下

![image](https://user-images.githubusercontent.com/10209135/100370616-80645a80-3041-11eb-8604-7478f0b35ed0.png)

#### VisualVM-多合一故障处理工具

VisualVM，All-in-One Java Troubleshooting Tool，是一款功能强大的运行监视和故障处理工具（带图形界面），作为一个合格的JAVA程序员，应熟悉之

Oracle曾在VisualVM的软件说明中写上了“All-in-One”的字样，预示着它除了常规的运行监视、故障处理外，还将提供其他方面的能力，譬如性能分析（Profiling）。VisualVM的性能分析功能比起JProfiler、YourKit等专业且收费的Profiling工具都不遑多让。而且相比这些第三方工具，VisualVM还有一个很大的优点：不需要被监视的程序基于特殊Agent去运行，因此它的通用性很强，对应用程序实际性能的影响也较小，使得它可以直接应用在生产环境中。这个优点是JProfiler、YourKit等工具无法与之媲美的

VisualVM 包含了所有基础故障处理工具的功能，且可视化
- 显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）
- 监视程序应用的CPU、GC、堆、方法区以及线程的信息（jstat、jstack）
- 生成及分析堆转储快照（jmap、jhat）
- 方法级的程序运行性能分析，找出被调用最多、运行时间最长的方法
- 离线程序快照；收集程序的运行时配置、线程dump、内存dump等信息建立一个快照，可以将快照发送开发者处进行 Bug 反馈
- 结合其他 plugins 做到无限可能性 ...

命令行执行 jvisualvm，点击左边 attch 到具体进程，可以得到如下结果

（需要安装插件VisualGC，左上角点击工具 - 插件 - 可用插件 - VisualGC，点击安装，完成后就可以点击VisualGC界面了，该界面对于分析非常有用）

![image](https://user-images.githubusercontent.com/10209135/100355624-0cb85280-302d-11eb-975e-f8ea194fad8d.png)

#### Arthas-Alibaba开源的Java诊断工具

Alibaba 开源的工具，Java应用诊断利器，中文文档：https://arthas.aliyun.com/zh-cn/ （附带安装文档），关于 Arthas 如何使用，各命令参数的含义等，建议直接查看中文文档

由于许多线上业务都不会带有图形界面，只能用命令行，那么使用 VisualVM 分析需要远程连接，服务端需要开新端口，存在风险安全问题，而 Arthas 可以支持命令行层面的调试，但功能强大，完全不逊色于 VisualVM。总体来说，Arthas 使用起来要稍微复杂一些，毕竟需要输入许多参数，有些命令如 sc、stack，要求输入 class-pattern、method-pattern、field-pattern，在这一点上不如 VisualVM 方便，但支持的功能点更精细化

简单举例Arthas的功能
- sc、sm：列举所有被JVM加载的类和方法
- classloader：列举所有类加载器名称，实例个数，类加载次数
- jad：反编译类文件为JAVA代码（确认代码是否发布成功）
- getstatic：获取类变量详细信息
- thread：打印线程信息，线程堆栈信息
- jvm：打印JVM详细参数
- dashboard：间隔输出监控信息，包含线程状态，线程占CPU比率，各区域内存使用情况、运行时参数
- heapdump：生成堆转储快照（dumpw文件）
- 还有很多linux式命令，如kepmap、cls、history、cat、echo、pwd、grep、tee、stop

Windows上使用方法
- 通过 jps 获取 pid，执行 as < pid >，得到如下结果
```cmd
as 13872
JAVA_HOME: C:\Program Files\Java\jdk1.8.0_231
telnet port: 3658
http port: 8563
INFO: Could not find files for the given pattern(s).
telnet wasn't found, please google how to install telnet under windows.
Try to visit http://127.0.0.1:8563 to connecto arthas server.
```
- 浏览器访问 http://127.0.0.1:8563 ，可以打开一个命令行界面，输入help，得到如下结果

![image](https://user-images.githubusercontent.com/10209135/100407826-6f006a00-30a4-11eb-8764-2f80358c3e9d.png)

### 性能调优记录

下面主要从多个角度做下性能调优记录，有一些是大神们总结的宝贵经验，有一些是面试常见问题，同时会探索 VisualVM 和 Arthas 可否支持这个具体的功能，若可以，就记录下来

#### 程序死循环问题排查

解决问题：程序卡主不动了，可能出现死循环，要检测出来

测试代码

```java
package thread;

public class DeadLoop {

    public static void main(String[] args) {
        test(0);
    }

    static void test(int x) {
        while (true);
    }
}
```

jstack
- 通过jps查到pid
- 执行jstack pid
- 找到正在运行不停的线程，怀疑可能是死循环线程
```cmd
"main" #1 prio=5 os_prio=0 tid=0x00000000035b3800 nid=0x4788 runnable [0x00000000033af000]
   java.lang.Thread.State: RUNNABLE
        at thread.DeadLoop.test(DeadLoop.java:10)
        at thread.DeadLoop.main(DeadLoop.java:6)
```

VisualVM
- 执行jvisualvm
- 左侧点击具体线程，点击抽样器，点击CPU，进行分析，过段时间后点击停止
- 可以看到CPU占用最多的线程，这里是 main

![image](https://user-images.githubusercontent.com/10209135/100446536-f1f7e380-30e9-11eb-97f6-53595c9824c0.png)

- 点击线程，点击线程Dump，可以看到 main 线程的堆栈信息，然后查看具体业务代码

![image](https://user-images.githubusercontent.com/10209135/100446863-83675580-30ea-11eb-965e-65d1eeda9a6c.png)


Arthas
- 通过jps查到pid
- 执行as < pid >，登陆 localhost
- 执行dashboard，可以看到main线程占CPU100%

![image](https://user-images.githubusercontent.com/10209135/100447286-4780c000-30eb-11eb-9e04-e61bdebbf0be.png)

- main线程的ID为1，执行 thread 1，可以看到线程堆栈信息，然后查看具体业务代码

![image](https://user-images.githubusercontent.com/10209135/100447395-7bf47c00-30eb-11eb-9d8c-aeceb2ce1bb4.png)

#### 程序死锁问题排查

解决问题：程序卡主不动了，可能出现死锁，要检测出来

测试代码

```java
package thread;

public class DeadThread {

    private static String a = "a";
    private static String b = "b";

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            synchronized (a) {
                System.out.println("threadA进入a同步块，执行中...");
                try {
                    Thread.sleep(2000);
                    synchronized (b) {
                        System.out.println("threadA进入b同步块，执行中...");
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "threadA");

        Thread threadB = new Thread(() -> {
           synchronized (b) {
               System.out.println("threadB进入b同步块，执行中...");
               synchronized (a) {
                   System.out.println("threadB进入a同步块，执行中...");
               }
           }
        }, "threadB");

        threadA.start();
        threadB.start();
    }
}
```

输出结果（死锁状态）
```
threadA进入a同步块，执行中...
threadB进入b同步块，执行中...
...
```

jstack
- 通过jps查到pid
- 执行jstack pid
```cmd
Found one Java-level deadlock:
=============================
"threadB":
  waiting to lock monitor 0x00000000031cb738 (object 0x000000076b16d6a0, a java.lang.String),
  which is held by "threadA"
"threadA":
  waiting to lock monitor 0x00000000031ce078 (object 0x000000076b16d6d0, a java.lang.String),
  which is held by "threadB"
```

VisualVM
- 执行jvisualvm
- 点击线程，可以看到“检测到死锁！”字眼

![image](https://user-images.githubusercontent.com/10209135/100403802-c8fc3200-309a-11eb-8c5c-7125ec58f025.png)

- 点击线程Dump生成快照，可以查到堆栈信息

![image](https://user-images.githubusercontent.com/10209135/100403851-ec26e180-309a-11eb-88d0-f05e606a1055.png)

Arthas
- 通过jps查到pid
- 执行as < pid >，登陆 localhost
- thread -b，可以检测死锁

![image](https://user-images.githubusercontent.com/10209135/100404838-52146880-309d-11eb-82b5-e0926154ad78.png)

MAT
- 从 https://www.eclipse.org/mat/downloads.php 中下载对应操作系统版本的软件
- 打开软件后，导入 dump 文件

#### 生成与分析Dump文件

解决问题：在线生成堆Dump和线程Dump进行分析，若可以，支持导入dump文件进行分析

生成dump文件有多种方法
- jmap、VisualVM、Arthas 都可以生成 dump 文件
- 通过参数 -XX:+HeapDumpOnOutOfMemoryError，-XX:HeapDumpPath=，在程序OOM时生成 dump 文件
- 通过参数 -XX:+HeapDumpOnCtrlBeak，加 ctrl + break 键，主动生成 dump 文件
- Linux系统下，通过 kill -3 命令发送进程退出信号“吓唬”一下虚拟机，也能拿到 dump 文件

分析dump文件一般用 MAT 或者 VisualVM

MAT
- 从 https://www.eclipse.org/mat/downloads.php 中下载对应操作系统版本的 MAT
- （左上角点击 File - Open Heap Dump，导入dump文件）
- 点击 Histogram、Dominator Tree、Leak Suspects 等可以进行进一步分析

![image](https://user-images.githubusercontent.com/10209135/103257164-1554cd80-49cb-11eb-8a50-ed9af9cc79a2.png)

VisualVM
- 可以用来分析dump文件（可以通过jmap生成）
- （左上角点击 文件 - 装入，导入dump文件，然后分析）
- 还支持对正在运行的线程生成堆转储快照、线程堆栈快照
- （左边点击一个具体线程，左上角点击 应用程序 - 堆Dump 或者 线程Dump）
- 下面是堆Dump的一个分析界面，它其中除了支持显示 概要、类、实例数 等详细信息外，还有一个OQL控制台功能，可以使用 sql 语句来查询堆中对象

![image](https://user-images.githubusercontent.com/10209135/100372246-ef42b300-3043-11eb-904d-2af924a86f8b.png)

Arthas
- heapdump命令指南

![image](https://user-images.githubusercontent.com/10209135/100407109-82aad100-30a2-11eb-912a-05f8d2bc95b5.png)

- 执行heapdump C:\Users\Admin\Desktop\1.hprof，会生成一份文件在桌面

![image](https://user-images.githubusercontent.com/10209135/100407576-b5a19480-30a3-11eb-95d7-dcf4db686a1a.png)

- 然后通过VisualVM进行分析（Arthas不支持分析dump文件，参考：https://github.com/alibaba/arthas/issues/806）

#### OOM异常问题排查

参考《深入理解JAVA虚拟机》第二版2.4.1

通过参数 -XX:+HeapDumpOnOutOfMemoryError 可以让虚拟机出现内存溢出异常（java.lang.OutOfMemoryError）时，Dump出当前的内存堆转储快照以便事后进行分析

要解决这个区域的异常，一般的手段是先通过内存映像分析工具（如Eclipse Memory Analyzer、VisualVM）对Dump出来的堆转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是先分清楚是出现了 内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）

如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链。于是就能找到泄漏对象是通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收它们的。掌握了泄漏对象的类型信息以及GC Roots引用链的信息，就可以比较准确地定位出泄漏代码的位置

如果不存在泄漏，换句话说，就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗

下面我们来模拟一次

- 引用 [运行时数据区域.md#例子1-创建过多对象导致堆区溢出](运行时数据区域.md#例子1-创建过多对象导致堆区溢出) 中的方法，生成了一份 dump 文件
- 打开 MAT 工具，导入 dump 文件
- 在 Overview 中点击 Leak Suspects，再点击 Details，可以看到许多分析点
- （通过 All Accumulated Objects by Class 可以查看导致OOM的实例）
- （通过 Thread Stack 可以查看导致OOM的代码堆栈信息）
![image](https://user-images.githubusercontent.com/10209135/103257486-58637080-49cc-11eb-8bd3-c270d87500dc.png)
![image](https://user-images.githubusercontent.com/10209135/103257496-60231500-49cc-11eb-828f-389582ac640f.png)
![image](https://user-images.githubusercontent.com/10209135/103257479-53062600-49cc-11eb-828c-11553c41f154.png)
- 回到 Overview，点击 dominator_tree，选择最多数量的实例
- 右键选择 Path To GC Roots，选择不同的 exclude 可以查看 GC Roots 中强软弱虚引用的信息，从而进一步确认是内存泄漏还是内存溢出
![image](https://user-images.githubusercontent.com/10209135/103257615-dd4e8a00-49cc-11eb-94cb-4332ca959f2a.png)
![image](https://user-images.githubusercontent.com/10209135/103257692-34545f00-49cd-11eb-8030-edb8323d42ce.png)

#### 线上业务日志切面

解决问题：经常遇到程序出现问题，但排查错误的一些必要信息，譬如方法参数、返回值等，在开发时没有打印到日志中，以至于不得不停掉服务，通过调试增量来加入日志代码以解决问题。当遇到生产环境服务无法随便停止时，缺一两句日志导致排错进行不下去是一件非常郁闷的事情。需要一种能动态添加日志跟踪的功能，类似于Spring AOP切面

假设有这么一段代码
```java
package thread;

public class DeadLoop {

    public static void main(String[] args) {
        test(0);
    }

    static void test(int x) {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        test(x + 1);
    }
}
```

VisualVM
- 安装BTrace
- 左侧点击具体线程，右键选择框里点击Trace application...
- 在代码框中输入以下代码
```java
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

@BTrace
public class TracingScript {
    @OnMethod(
        clazz="thread.DeadLoop", method="test"
    )
    public static void func(@Self thread.DeadLoop instance, int x) {
        println(strcat("方法参数：", str(x)));
    }
}
```
- （如果方法是有类型的返回，比如 int类型，则@OnMethod中添加location=@Location(Kind.RETURN)，func参数中添加@Return int result）
- 点击start，可以看到以下输出，能打印出方法参数

![image](https://user-images.githubusercontent.com/10209135/100411396-9e1ad980-30ac-11eb-8ba6-60794175f417.png)

### CPU占用过高问题排查

这是一道经典的面试问题

解决办法思路：（1）查到占CPU高的进程和线程ID（2）通过线程堆栈工具查找堆栈日志（3）分析业务代码

关于（1），可以用 VisualVM、Arthas，而 Linux 上还可以用 top 命令

关于（2），可以用 jstack、VisualVM、Arthas

以Linux上举例
- top，查到占CPU最高的进程ID
- top -H -p < 进程ID >，查到进程中占CPU最高的线程ID
- 将线程ID从十进制转十六进制
- jstack 进程ID | grep 十六进制线程ID -A 30，查看线程堆栈，可定位到哪一行代码导致的

CPU占用过高、死循环、死锁问题，都可以用 VisualVM、Arthas 来分析

### 亿级流量系统调优案例

参考：https://github.com/peteryuanpan/notebook/issues/140

TODO

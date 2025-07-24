---
slug: alibaba-java-arthas-diagnostic-tool
title: Alibaba Java诊断利器Arthas
authors: [ledai]
tags: [学习历程, Arthas]
date: 2018-07-18
---
  <h1>Alibaba Java诊断利器Arthas</h1>
今天无意间发现了一个alibaba的开源jvm分析利器

<!-- truncate -->

刚刚开源的阿里内部广泛使用的Java在线诊断工具Arthas，今天来到了Github trending榜首，相当于程序员的微博热搜榜首！欢迎大家关注、试用和反馈：https://github.com/alibaba/arthas


<h2>Arthas 能为你做什么？</h2>

当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
是否有一个全局视角来查看系统的运行状况？
有什么办法可以监控到JVM的实时运行状态？
Arthas 是基于 Greys 进行二次开发的全新在线诊断工具，利用Java6的Instrumentation特性，动态增强你所指定的类，获取你想要到的信息, 采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，让你在定位、分析诊断问题时看每一个操作都看起来是那么的 666

<h2>安装</h2>

Arthas 支持在 Linux/Unix/Mac 等平台上一键安装，请复制以下内容，并粘贴到命令行中，敲 回车 执行即可：

`curl -L https://alibaba.github.io/arthas/install.sh | sh`
Windows在Download栏下载最新的 bin.zip 包，解压后在bin目录有 as.bat。此脚本暂时只接受一个参数 pid，即只能诊断本机上的 Java 进程。

Dashboard

https://alibaba.github.io/arthas/dashboard


Web Console

https://alibaba.github.io/arthas/web-console


<h2>命令列表</h2>

`dashboard`

当前系统的实时数据面板，按 ctrl+c 退出。
当运行在Ali-tomcat时，会显示当前tomcat的实时信息，如HTTP请求的qps, rt, 错误数, 线程池信息等等。
`thread`

查看当前线程信息，查看线程的堆栈
cpu占比是如何统计出来的？
这里的cpu统计的是，一段采样间隔内，当前JVM里各个线程所占用的cpu时间占总cpu时间的百分比。其计算方法为： 首先进行一次采样，获得所有线程的cpu的使用时间(调用的是java.lang.management.ThreadMXBean#getThreadCpuTime这个接口)，然后睡眠一段时间，默认100ms，可以通过-i参数指定，然后再采样一次，最后得出这段时间内各个线程消耗的cpu时间情况，最后算出百分比。
注意： 这个统计也会产生一定的开销（JDK这个接口本身开销比较大），因此会看到as的线程占用一定的百分比，为了降低统计自身的开销带来的影响，可以把采样间隔拉长一些，比如5000毫秒。
如果想看从Java进程启动开始到现在的cpu占比情况：可以使用show-busy-java-threads这个脚本
`jvm`

查看当前JVM信息
THREAD相关
COUNT: JVM当前活跃的线程数
DAEMON-COUNT: JVM当前活跃的守护线程数
LIVE-COUNT: 从JVM启动开始曾经活着的最大线程数
STARTED-COUNT: 从jvm启动开始总共启动过的线程次数
`sysprop`

查看当前JVM的系统属性(System Property)
`getstatic`

通过getstatic命令可以方便的查看类的静态属性。使用方法为getstatic class_name field_name
如果该静态属性是一个复杂对象，还可以支持在该属性上通过ognl表示进行遍历，过滤，访问对象的内部属性等操作。
`sc`

查看JVM已加载的类信息
`sm`

查看已加载类的方法信息
“Search-Method” 的简写，这个命令能搜索出所有已经加载了 Class 信息的方法信息。
sm 命令只能看到由当前类所声明 (declaring) 的方法，父类则无法看到。
`dump`

dump 已加载类的 bytecode 到特定目录
`jad`

反编译指定已加载类的源码
jad 命令将 JVM 中实际运行的 class 的 byte code 反编译成 java 代码，便于你理解业务逻辑；
在 Arthas Console 上，反编译出来的源码是带语法高亮的，阅读更方便
当然，反编译出来的 java 代码可能会存在语法错误，但不影响你进行阅读理解
`classloader`

查看classloader的继承树，urls，类加载信息
classloader 命令将 JVM 中所有的classloader的信息统计出来，并可以展示继承树，urls等。
可以让指定的classloader去getResources，打印出所有查找到的resources的url。对于ResourceNotFoundException比较有用。
`redefine`

加载外部的.class文件，redefine jvm已加载的类。
注意， redefine后的原来的类不能恢复，redefine有可能失败（比如增加了新的field），参考jdk本身的文档。
`monitor`

方法执行监控
对匹配 class-pattern／method-pattern的类、方法的调用进行监控。
monitor 命令是一个非实时返回命令，实时返回命令是输入之后立即返回，而非实时返回的命令，则是不断的等待目标 Java 进程返回信息，直到用户输入 Ctrl+C 为止。服务端是以任务的形式在后台跑任务，植入的代码随着任务的中止而被不会被执行，所以任务关闭后，不会对原有性能产生太大影响，而且原则上，任何 Arthas 的命令也不会引起任何原有业务逻辑的改变。
`watch`

方法执行数据观测
让你能方便的观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写 OGNL 表达式进行对应变量的查看。
`trace`

方法内部调用路径，并输出方法路径上的每个节点上耗时
trace 命令能主动搜索 class-pattern／method-pattern 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。
`stack`

输出当前方法被调用的调用路径
很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多，或者你根本就不知道这个方法是从那里被执行了，此时你需要的是 stack 命令。
`tt`

方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测
watch 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。
这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。
于是乎，TimeTunnel 命令就诞生了。
`options`

全局开关

进阶使用

基础命令

help——查看命令帮助信息
cls——清空当前屏幕区域
session——查看当前会话的信息
reset——重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
version——输出当前目标 Java 进程所加载的 Arthas 版本号
quit——退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
shutdown——关闭 Arthas 服务端，所有 Arthas 客户端全部退出
keymap——Arthas快捷键列表及自定义快捷键

jvm相关

dashboard——当前系统的实时数据面板
thread——查看当前 JVM 的线程堆栈信息
jvm——查看当前 JVM 的信息
sysprop——查看和修改JVM的系统属性
New! getstatic——查看类的静态属性

class/classloader相关

sc——查看JVM已加载的类信息
sm——查看已加载类的方法信息
dump——dump 已加载类的 byte code 到特定目录
redefine——加载外部的.class文件，redefine到JVM里
jad——反编译指定已加载类的源码
classloader——查看classloader的继承树，urls，类加载信息，使用classloader去getResource

monitor/watch/trace相关

请注意，这些命令，都通过字节码增强技术来实现的，会在指定类的方法中插入一些切面来实现数据统计和观测，因此在线上、预发使用时，请尽量明确需要观测的类、方法以及条件，诊断结束要执行 shutdown 或将增强过的类执行 reset 命令。
monitor——方法执行监控
watch——方法执行数据观测
trace——方法内部调用路径，并输出方法路径上的每个节点上耗时
stack——输出当前方法被调用的调用路径
tt——方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测

options

options——查看或设置Arthas全局开关

管道

Arthas支持使用管道对上述命令的结果进行进一步的处理，如sm org.apache.log4j.Logger | grep

grep——搜索满足条件的结果
plaintext——将命令的结果去除颜色
wc——按行统计输出结果

后台异步任务

当线上出现偶发的问题，比如需要watch某个条件，而这个条件一天可能才会出现一次时，异步后台任务就派上用场了

使用 > 将结果重写向到日志文件，使用 & 指定命令是后台运行，session断开不影响任务执行（生命周期默认为1天）
jobs——列出所有job
kill——强制终止任务
fg——将暂停的任务拉到前台执行
bg——将暂停的任务放到后台执行

Web Console

通过websocket连接Arthas。
Web Console

其他特性

异步命令支持
执行结果存日志
批处理的支持
ognl表达式的用法说明

进阶案例

【Arthas问题排查集】谁调用了System.exit/System.gc?
https://github.com/alibaba/arthas/issues/20
【Arthas问题排查集】活用ognl表达式
https://github.com/alibaba/arthas/issues/11
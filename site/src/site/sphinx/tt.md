tt
===

> 方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测

`watch` 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。

这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。

于是乎，TimeTunnel 命令就诞生了。

### 记录方法的调用

- 基本用法

  对于一个最基本的使用来说，就是记录下当前方法的每次调用环境现场。
  
  ```java
  $ tt -t -n 3 *Test print
  Press Ctrl+D or Ctrl+X to abort.
  Affect(class-cnt:1 , method-cnt:1) cost in 115 ms.
  +----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
  |    INDEX |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
  +----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
  |     1007 |  2015-07-26 12:23:21 |        138 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
  +----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
  |     1008 |  2015-07-26 12:23:22 |        143 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
  +----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
  |     1009 |  2015-07-26 12:23:23 |        130 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
  +----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
  $ 
  ```

- 命令参数解析

  - `-t`

     tt 命令有很多个主参数，`-t` 就是其中之一。这个参数的表明希望记录下类 `*Test` 的 `print` 方法的每次执行情况。
  
  - `-n 3`

     当你执行一个调用量不高的方法时可能你还能有足够的时间用 `CTRL+C` 中断 tt 命令记录的过程，但如果遇到调用量非常大的方法，瞬间就能将你的 JVM 内存撑爆。
     
     此时你可以通过 `-n` 参数指定你需要记录的次数，当达到记录次数时 Arthas 会主动中断tt命令的记录过程，避免人工操作无法停止的情况。

- 表格字段说明

|表格字段|字段解释|
|---|---|
|INDEX|时间片段记录编号，每一个编号代表着一次调用，后续tt还有很多命令都是基于此编号指定记录操作，非常重要。|
|TIMESTAMP|方法执行的本机时间，记录了这个时间片段所发生的本机时间|
|COST(ms)|方法执行的耗时|
|IS-RET|方法是否以正常返回的形式结束|
|IS-EXP|方法是否以抛异常的形式结束|
|OBJECT|执行对象的`hashCode()`，注意，曾经有人误认为是对象在JVM中的无力内存地址，但很遗憾他不是。但他能帮助你简单的标记当前执行方法的类实体|
|CLASS|执行的类名|
|METHOD|执行的方法名|

- 条件表达式

    不知道大家是否有在使用过程中遇到以下困惑
	* Arthas 似乎很难区分出重载的方法
	* 我只需要观察特定参数，但是 tt 却全部都给我记录了下来
    
    条件表达式也是用 `OGNL` 来编写，核心的判断对象依然是 `Advice` 对象。除了 `tt` 命令之外，`watch`、`trace`、`stack` 命令也都支持条件表达式。
  
- 解决方法重载

     `tt -t *Test print params[0].length==1`
     
     通过制定参数个数的形式解决不同的方法签名，如果参数个数一样，你还可以这样写
     
     `tt -t *Test print 'params[1] instanceof Integer'`
  
- 解决指定参数

     `tt -t *Test print params[0].mobile=="13989838402"`

- 构成条件表达式的 `Advice` 对象

    前边看到了很多条件表达式中，都适用了 `params[0]`，有关这个变量的介绍，请参考[表达式核心变量](advice-class.md)

### 检索调用记录

当你用 `tt` 记录了一大片的时间片段之后，你希望能从中筛选出自己需要的时间片段，这个时候你就需要对现有记录进行检索。

假设我们有这些记录

```
$ tt -l
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1000 |  2015-07-26 01:16:27 |        130 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1001 |  2015-07-26 01:16:27 |          0 |    false |     true |      0x42cc13a0 |                GaOgnlUtilsTest |                   printAddress |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1002 |  2015-07-26 01:16:28 |        119 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1003 |  2015-07-26 01:16:28 |          0 |    false |     true |      0x42cc13a0 |                GaOgnlUtilsTest |                   printAddress |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1004 |  2015-07-26 12:21:56 |        130 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1005 |  2015-07-26 12:21:57 |        138 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1006 |  2015-07-26 12:21:58 |        130 |     true |    false |      0x42cc13a0 |                GaOgnlUtilsTest |                          print |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
Affect(row-cnt:7) cost in 2 ms.
$ 
```

我需要筛选出 `printAddress` 方法的调用信息

```
$ tt -s method.name=="printAddress"
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1001 |  2015-07-26 01:16:27 |          0 |    false |     true |      0x42cc13a0 |                GaOgnlUtilsTest |                   printAddress |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1003 |  2015-07-26 01:16:28 |          0 |    false |     true |      0x42cc13a0 |                GaOgnlUtilsTest |                   printAddress |
+----------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
Affect(row-cnt:2) cost in 55 ms.
$ 
```

你需要一个 `-s` 参数。<span style="color:red;">同样的，搜索表达式的核心对象依旧是 `Advice` 对象。</span>

### 查看调用信息

对于具体一个时间片的信息而言，你可以通过 `-i` 参数后边跟着对应的 `INDEX` 编号查看到他的详细信息。

```
$ 
$ tt -i 1003
+-----------------+------------------------------------------------------------------------------------------------------+
|           INDEX | 1003                                                                                                 |
+-----------------+------------------------------------------------------------------------------------------------------+
|      GMT-CREATE | 2015-07-26 01:16:28                                                                                  |
+-----------------+------------------------------------------------------------------------------------------------------+
|        COST(ms) | 0                                                                                                    |
+-----------------+------------------------------------------------------------------------------------------------------+
|          OBJECT | 0x42cc13a0                                                                                           |
+-----------------+------------------------------------------------------------------------------------------------------+
|           CLASS | GaOgnlUtilsTest                                                                                      |
+-----------------+------------------------------------------------------------------------------------------------------+
|          METHOD | printAddress                                                                                         |
+-----------------+------------------------------------------------------------------------------------------------------+
|       IS-RETURN | false                                                                                                |
+-----------------+------------------------------------------------------------------------------------------------------+
|    IS-EXCEPTION | true                                                                                                 |
+-----------------+------------------------------------------------------------------------------------------------------+
|   PARAMETERS[0] | Address@53448f87                                                                                     |
+-----------------+------------------------------------------------------------------------------------------------------+
| THROW-EXCEPTION | java.lang.RuntimeException: test                                                                     |
|                 |     at GaOgnlUtilsTest.printAddress(Unknown Source)                                                  |
|                 |     at GaOgnlUtilsTest.<init>(Unknown Source)                                                        |
|                 |     at GaOgnlUtilsTest.main(Unknown Source)                                                          |
+-----------------+------------------------------------------------------------------------------------------------------+
Affect(row-cnt:1) cost in 1 ms.
$ 
```

### 重做一次调用

当你稍稍做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。

`tt` 命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个 `INDEX` 编号的时间片自主发起一次调用，从而解放你的沟通成本。此时你需要 `-p` 参数。

```
$ tt -i 1003 -p
+-----------------+---------------------------------------------------------------------------------------------------------+
|        RE-INDEX | 1003                                                                                                    |
+-----------------+---------------------------------------------------------------------------------------------------------+
|      GMT-REPLAY | 2015-07-26 17:29:51                                                                                     |
+-----------------+---------------------------------------------------------------------------------------------------------+
|          OBJECT | 0x42cc13a0                                                                                              |
+-----------------+---------------------------------------------------------------------------------------------------------+
|           CLASS | GaOgnlUtilsTest                                                                                         |
+-----------------+---------------------------------------------------------------------------------------------------------+
|          METHOD | printAddress                                                                                            |
+-----------------+---------------------------------------------------------------------------------------------------------+
|   PARAMETERS[0] | Address@53448f87                                                                                        |
+-----------------+---------------------------------------------------------------------------------------------------------+
|       IS-RETURN | false                                                                                                   |
+-----------------+---------------------------------------------------------------------------------------------------------+
|    IS-EXCEPTION | true                                                                                                    |
+-----------------+---------------------------------------------------------------------------------------------------------+
| THROW-EXCEPTION | java.lang.RuntimeException: test                                                                        |
|                 |     at GaOgnlUtilsTest.printAddress(GaOgnlUtilsTest.java:78)                                            |
|                 |     at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)                                      |
|                 |     at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)                    |
|                 |     at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)            |
|                 |     at java.lang.reflect.Method.invoke(Method.java:483)                                                 |
|                 |     at com.github.ompc.Arthas.util.GaMethod.invoke(GaMethod.java:81)                                     |
|                 |     at com.github.ompc.Arthas.command.TimeTunnelCommand$6.action(TimeTunnelCommand.java:592)             |
|                 |     at com.github.ompc.Arthas.server.DefaultCommandHandler.execute(DefaultCommandHandler.java:175)       |
|                 |     at com.github.ompc.Arthas.server.DefaultCommandHandler.executeCommand(DefaultCommandHandler.java:83) |
|                 |     at com.github.ompc.Arthas.server.GaServer$4.run(GaServer.java:329)                                   |
|                 |     at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)                  |
|                 |     at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)                  |
|                 |     at java.lang.Thread.run(Thread.java:745)                                                            |
+-----------------+---------------------------------------------------------------------------------------------------------+
replay time fragment[1003] success.
Affect(row-cnt:1) cost in 3 ms.
$ 
```

你会发现结果虽然一样，但调用的路径发生了变化，有原来的程序发起变成了 Arthas 自己的内部线程发起的调用了。

- 需要强调的点

  1. **ThreadLocal 信息丢失**

     很多框架偷偷的将一些环境变量信息塞到了发起调用线程的 ThreadLocal 中，由于调用线程发生了变化，这些 ThreadLocal 线程信息无法通过 Arthas 保存，所以这些信息将会丢失。
     
     一些常见的 CASE 比如：鹰眼的 TraceId 等。
  
  2. **引用的对象**

     需要强调的是，`tt` 命令是将当前环境的对象引用保存起来，但仅仅也只能保存一个引用而已。如果方法内部对入参进行了变更，或者返回的对象经过了后续的处理，那么在 `tt` 查看的时候将无法看到当时最准确的值。这也是为什么 `watch` 命令存在的意义。
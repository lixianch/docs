## GC垃圾回收机制  
 
在学习GC之前，你首先应该记住一个单词：“stop-the-world”。Stop-the-world会在任何一种GC算法中发生。Stop-the-world意味着 JVM 因为要执行GC而停止了应用程序的执行。当Stop-the-world发生时，除了GC所需的线程以外，所有线程都处于等待状态，直到GC任务完成。GC优化很多时候就是指减少Stop-the-world发生的时间。  
**按代的垃圾回收机制**  
在Java程序中不能显式地分配和注销内存。有些人把相关的对象设置为null或者调用System.gc()来试图显式地清理内存。设置为null至少没什么坏处，但是调用System.gc()会显著地影响系统性能，必须彻底杜绝。  
在Java中，开发人员无法直接在程序代码中清理内存，而是由垃圾回收器自动寻找不必要的垃圾对象，并且清理掉他们。垃圾回收器会在下面两种假设（hypotheses）成立的情况下被创建（称之为假设不如改为推测（suppositions）或者前提（preconditions））。  
- 大多数对象会很快变得不可达  
- 只有很少的由老对象（创建时间较长的对象）指向新生对象的引用  

这些假设我们称之为弱年代假设（ weak generational hypothesis）。为了强化这一假设，HotSpot虚拟机将其物理上划分为两个–新生代（young generation）和老年代（old generation）。  
**新生代（Young generation）:** 绝大多数最新被创建的对象会被分配到这里，由于大部分对象在创建后会很快变得不可到达，所以很多对象被创建在新生代，然后消失。对象从这个区域消失的过程我们称之为”minor GC“。  
**老年代（Old generation）:** 对象没有变得不可达，并且从新生代中存活下来，会被拷贝到这里。其所占用的空间要比新生代多。也正由于其相对较大的空间，发生在老年代上的GC要比新生代少得多。对象从老年代中消失的过程，我们称之为”major GC“（或者”full GC“）  
请看下面这个图表。  
![](http://i.imgur.com/KNyhFej.png)
 图1 : GC 空间 & 数据流 
上图中的持久代（ **permanent generation** ）也被称为**方法区（method area）**。他用来保存类常量以及字符串常量。因此，这个区域不是用来永久的存储那些从老年代存活下来的对象。这个区域也可能发生GC。并且发生在这个区域上的GC事件也会被算为major GC。  
有些人可能会问：
**如果老年代的对象需要引用一个新生代的对象，会发生什么呢？**
为了解决这个问题，老年代中存在一个”**card table**“，他是一个512 byte大小的块。所有老年代的对象指向新生代对象的引用都会被记录在这个表中。当针对新生代执行GC的时候，只需要查询card table来决定是否可以被收集，而不用查询整个老年代。这个card table由一个write barrier来管理。**write barrier**给GC带来了很大的性能提升，虽然由此可能带来一些开销，但GC的整体时间被显著的减少。  
![](http://i.imgur.com/2wlCZKh.png)
图 2: Card Table 结构  
**新生代的构成**  
新生代是用来保存那些第一次被创建的对象，他可以被分为三个空间  
- 一个伊甸园空间（**Eden** ）  
- 两个幸存者空间（**Survivor** ）  

一共有三个空间，其中包含两个幸存者空间。每个空间的执行顺序如下：  
1. 绝大多数刚刚被创建的对象会存放在伊甸园空间。  
2. 在伊甸园空间执行了第一次GC之后，存活的对象被移动到其中一个幸存者空间。  
3.  此后，在伊甸园空间执行GC之后，存活的对象会被堆积在同一个幸存者空间。  
4.  当一个幸存者空间饱和，还在存活的对象会被移动到另一个幸存者空间。之后会清空已经饱和的那个幸存者空间。  
5.  在以上的步骤中重复几次依然存活的对象，就会被移动到老年代。  

如果你仔细观察这些步骤就会发现，其中一个幸存者空间必须保持是空的。如果两个幸存者空间都有数据，或者两个空间都是空的，那一定标志着你的系统出现了某种错误。  
通过频繁的minor GC将数据移动到老年代的过程可以用下图来描述：  
![](http://i.imgur.com/QAdF6eD.png)  
图 3: GC执行前后对比  
需要注意的是HotSpot虚拟机使用了两种技术来加快内存分配。他们分别是是”**bump-the-pointer**“和“**TLABs（Thread-Local Allocation Buffers）**”。  
**Bump-the-pointer**技术跟踪在伊甸园空间创建的最后一个对象。这个对象会被放在伊甸园空间的顶部。如果之后再需要创建对象，只需要检查伊甸园空间是否有足够的剩余空间。如果有足够的空间，对象就会被创建在伊甸园空间，并且被放置在顶部。这样以来，每次创建新的对象时，只需要检查最后被创建的对象。这将极大地加快内存分配速度。但是，如果我们在多线程的情况下，事情将截然不同。如果想要以线程安全的方式以多线程在伊甸园空间存储对象，不可避免的需要加锁，而这将极大地的影响性能。**TLABs** 是HotSpot虚拟机针对这一问题的解决方案。该方案为每一个线程在伊甸园空间分配一块独享的空间，这样每个线程只访问他们自己的TLAB空间，再与bump-the-pointer技术结合可以在不加锁的情况下分配内存。  
请务必记住在对象刚刚被创建之后，是保存在伊甸园空间的。那些长期存活的对象会经由幸存者空间转存在老年代空间。  
**老年代GC处理机制**  
老年代空间的GC事件基本上是在空间已满时发生，执行的过程根据GC类型不同而不同，因此，了解不同的GC类型将有助于你理解本节的内容。  
JDK7一共有5种GC类型：  
1. Serial GC  
2. Parallel GC  
3. Parallel Old GC (Parallel Compacting GC)  
4. Concurrent Mark & Sweep GC  (or “CMS”)  
5. Garbage First (G1) GC  

其中，Serial GC不应该被用在服务器上。这种GC类型在单核CPU的桌面电脑时代就存在了。使用Serial GC会显著的降低应用的性能指标。  
现在，让我们共同学习每一种GC类型  
**1. Serial GC (-XX:+UseSerialGC)**  
新生代空间的GC方式我们在前面已经介绍过了，在老年代空间中的GC采取称之为”mark-sweep-compact“的算法。  
1.算法的第一步是标记老年代中依然存活对象。（标记）  
2.第二步，从头开始检查堆内存空间，并且只留下依然幸存的对象。（清理）  
最后一步，从头开始，顺序地填满堆内存空间，并且将对内存空间分成两部分：一个保存着对象，另一个空着（压缩）。  
**2. Parallel GC (-XX:+UseParallelGC)**  
![](http://i.imgur.com/cmQMnbq.png)  
图 4: Serial GC 与 Parallel GC的区别  
从上图中，你可以轻易地看出serial GC和parallel GC的区别，serial GC只使用一个线程执行GC，而parallel GC使用多个线程，因此parallel GC更高效。这种GC在内存充足以及多核的情况下会很有用，因此我们也称之为”**throughput GC**“。  
**3. Parallel Old GC(-XX:+UseParallelOldGC)**  
Parallel Old GC在JDK5之后出现。与parallel GC相比，唯一的区别在于针对老年代的GC算法。Parallel Old GC分为三步：标记-汇总-压缩（mark – summary – compaction）。汇总（summary）步骤与清理（sweep）的不同之处在于，其将依然幸存的对象分发到GC预先处理好的不同区域，算法相对清理来说略微复杂一点。  
**4. CMS GC (-XX:+UseConcMarkSweepGC)**  
![](http://i.imgur.com/B4JySce.png)  
图 5: Serial GC & CMS GC  
就像你从上图看到的那样, CMS GC比我之前解释的各种算法都要复杂很多。第一步初始化标记（initial mark） 比较简单。这一步骤只是查找那些距离类加载器最近的幸存对象。因此，停顿的时间非常短暂。在之后的并行标记（ concurrent mark ）步骤，所有被幸存对象引用的对象会被确认是否已经被追踪和校验。这一步的不同之处在于，在标记的过程中，其他的线程依然在执行。在重新标记（remark）步骤，会再次检查那些在并行标记步骤中增加或者删除的与幸存对象引用的对象。最后，在并行交换（ concurrent sweep ）步骤，转交垃圾回收过程处理。垃圾回收工作会在其他线程的执行过程中展开。一旦采取了这种GC类型，由GC导致的暂停时间会极其短暂。CMS GC也被称为低延迟GC。它经常被用在那些对于响应时间要求十分苛刻的应用之上。  
当然，这种GC类型在拥有stop-the-world时间很短的优点的同时，也有如下缺点：  
- 它会比其他GC类型占用更多的内存和CPU  
- 默认情况下不支持压缩步骤  

在使用这个GC类型之前你需要慎重考虑。如果因为内存碎片过多而导致压缩任务不得不执行，那么stop-the-world的时间要比其他任何GC类型都长，你需要考虑压缩任务的发生频率以及执行时间。  
**5. G1 GC**  
最后，我们来学习垃圾回收优先（G1）GC类型。  
![](http://i.imgur.com/wmbePaN.png)  
图 6:  G1 GC的结构  
如果你想要理解G1，首先你要忘记你所学过的新生代和老年代的概念。正如你在上图所看到的，每个对象被分配到不同的格子，随后GC执行。当一个区域装满之后，对象被分配到另一个区域，并执行GC。这中间不再有从新生代移动到老年代的三个步骤。这个类型是为了替代CMS GC而被创建的，因为CMS GC在长时间持续运作时会产生很多问题。  
G1最大的好处是性能，他比我们在上面讨论过的任何一种GC都要快。但是在JDK 6中，他还只是一个早期试用版本。在JDK7之后才由官方正式发布。就我个人看来，NHN在将JDK 7正式投入商用之前需要很长的一段测试期（至少一年）。因此你可能需要再等一段时间。并且，我也听过几次使用了JDK 6中的G1而导致Java虚拟机宕机的事件。请耐心的等到它更稳定吧。  
假如应用中创建的所有对象的大小和类型都是统一的，那么公司使用的WAS的GC参数可以是相同的。但是WAS所创建对象的大小和生命周期根据服务以及硬件的不同而不同。换句话说，不能因为某个应用使用的GC参数“A”，就说明同样的参数也能给其他服务带来最佳的效果。而是要因地制宜，有的放矢。我们需要找到适合每个WAS线程的参数，并且持续的监控和优化每个设备上的WAS实例。这并不是我的一家之谈，而是负责Oracle Java虚拟机研发的工程师在 JavaOne 2010上已经讨论过的。  
## GC监控  
**什么是GC监控？**  
**垃圾回收收集监控**指的是搞清楚JVM如何执行GC的过程，例如，我们可以查明：  
1.何时一个新生代中的对象被移动到老年代时，所花费的时间。  
2.Stop-the-world 何时发生的，持续了多长时间。  
GC监控是为了鉴别JVM是否在高效地执行GC，以及是否有必要进行额外的性能调优。基于以上信息，我们可以修改应用程序或者调整GC算法（GC优化）。  
**如何监控GC**  
有很多种方法可以监控GC，但其差别仅仅是GC操作通过何种方式展现而已。GC操作是由JVM来完成，而GC监控工具只是将JVM提供的GC信息展现给你，因此，不论你使用何种方式监控GC都将得到相同的结果。所以你也就不必去学习所有的监控GC的方法。但是因为学习每种监控方法不会占用太多时间，了解多一点可以帮助你根据不同的场景选择最为合适的方式。  
下面所列的工具以及JVM参数并不适用于所有的HVM供应商。这是因为并没有关于GC信息的强制标准。本文我们将使用HotSpot JVM (Oracle JVM)。  
首先，GC监控方法根据访问的接口不同，可以分成CUI 和GUI 两大类。CUI GC监控方法使用一个独立的叫做”jstat”的CUI应用，或者在启动JVM的时候选择JVM参数”verbosegc”。
GUI GC监控由一个单独的图形化应用来完成，其中三个最常用的应用是”jconsole”, “jvisualvm” 和 “Visual GC”。  
下面我们来详细学习每种方法。  
**jstat**  
jstat 是HotSpot JVM提供的一个监控工具。其他监控工具还有jps 和jstatd。有些时候，你可能需要同时使用三种工具来监控你的应用。jstat 不仅提供GC操作的信息，还提供类装载操作的信息以及运行时编译器操作的信息。本文将只涉及jstat能够提供的信息中与监控GC操作信息相关的功能。  
jstat 被放置在$JDK_HOME/bin。因此只要java 和 javac能执行，jstat 同样可以执行。  
你可以在命令行环境下执行如下语句。  
```
$> jstat –gc  $<vmid$> 1000
 
S0C       S1C       S0U    S1U      EC         EU          OC         OU         PC         PU         YGC     YGCT    FGC      FGCT     GCT
3008.0   3072.0    0.0     1511.1   343360.0   46383.0     699072.0   283690.2   75392.0    41064.3    2540    18.454    4      1.133    19.588
3008.0   3072.0    0.0     1511.1   343360.0   47530.9     699072.0   283690.2   75392.0    41064.3    2540    18.454    4      1.133    19.588
3008.0   3072.0    0.0     1511.1   343360.0   47793.0     699072.0   283690.2   75392.0    41064.3    2540    18.454    4      1.133    19.588
 
$>
```  
在上图的例子中，实际的数据会按照如下列输出：  
```
S0C    S1C     S0U     S1U    EC     EU     OC     OU     PC
```  
vmid (虚拟机 ID)，正如其名字描述的，它是虚拟机的ID，Java应用不论运行在本地还是远程的机器都会拥有自己独立的vmid。运行在本地机器上的vmid称之为lvmid (本地vmid)，通常是PID。如果想得到PID的值你可以使用ps命令或者windows任务管理器，但我们推荐使用jps来获取，因为PID和lvmid有时会不一致。jps 通过Java PS实现，jps命令会返回vmids和main方法的信息，正如ps命令展现PIDS和进程名字那样。  
首先通过jps命令找到你要监控的Java应用的vmid，并把它作为jstat的参数。当几个WAS实例运行在同一台设备上时，如果你只使用jps命令，将只能看到启动（bootstrap）信息。我们建议在这种情况下使用ps -ef | grep java与jps配合使用。  
想要得到GC性能相关的数据需要持续不断地监控，因此在执行jstat时，要规则地输出GC监控的信息。  
例如，执行”jstat –gc 1000″ (或 1s)会每隔一秒展示GC监控数据。”jstat –gc 1000 10″会每隔1秒展现一次，且一共10次。  

参数名称|描述
---|---
gc|输出每个堆区域的当前可用空间以及已用空间（伊甸园，幸存者等等），GC执行的总次数，GC操作累计所花费的时间。  
gccapactiy|输出每个堆区域的最小空间限制（ms）/最大空间限制（mx），当前大小，每个区域之上执行GC的次数。（不输出当前已用空间以及GC执行时间）。  
gccause|输出-gcutil提供的信息以及最后一次执行GC的发生原因和当前所执行的GC的发生原因  
gcnew|输出新生代空间的GC性能数据  
gcnewcapacity|输出新生代空间的大小的统计数据。  
gcold|输出老年代空间的GC性能数据。  
gcoldcapacity|输出老年代空间的大小的统计数据。  
gcpermcapacity|输出持久带空间的大小的统计数据。  
gcutil|输出每个堆区域使用占比，以及GC执行的总次数和GC操作所花费的事件。  

你可以只关心那些最常用的命令，你会经常用到 **-gcutil (或-gccause), -gc and –gccapacity**。  
-  -gcutil 被用于检查堆间的使用情况，GC执行的次数以及GC操作所花费的时间。  
-  -gccapacity以及其他的参数可以用于检查实际分配内存的大小。  

**使用-gc 参数你可以看到如下输出：**  
```
S0C      S1C    …   GCT
1248.0   896.0  …   1.246
1248.0   896.0  …   1.246
…        …      …   …
```  
不同的jstat参数输出不同类型的列，如下表所示，根据你使用的”jstat option”会输出不同列的信息。 

列|	说明	|Jstat参数
---|---|---
S0C|	输出Survivor0空间的大小。单位KB。|	-gc -gccapacity -gcnew -gcnewcapacity
S1C|	输出Survivor1空间的大小。单位KB。|	-gc -gccapacity -gcnew -gcnewcapacity
S0U|	输出Survivor0已用空间的大小。单位KB。|	-gc -gcnew
S1U|	输出Survivor1已用空间的大小。单位KB。|	-gc -gcnew
EC|	输出Eden空间的大小。单位KB。	|-gc -gccapacity -gcnew -gcnewcapacity
EU|	输出Eden已用空间的大小。单位KB。|	-gc -gcnew
OC|	输出老年代空间的大小。单位KB。|	-gc -gccapacity -gcold -gcoldcapacity
OU|	输出老年代已用空间的大小。单位KB。|	-gc -gcold
PC|	输出持久代空间的大小。单位KB。|	-gc -gccapacity -gcold -gcoldcapacity -gcpermcapacity
PU|	输出持久代已用空间的大小。单位KB。|	-gc -gcold
YGC|	新生代空间GC时间发生的次数。|	-gc -gccapacity -gcnew -gcnewcapacity -gcold -gcoldcapacity -gcpermcapacity -gcutil -gccause
YGCT|	新生代GC处理花费的时间。|	-gc -gcnew -gcutil -gccause
FGC|	full GC发生的次数。|	-gc -gccapacity -gcnew -gcnewcapacity -gcold -gcoldcapacity -gcpermcapacity -gcutil -gccause
FGCT|	full GC操作花费的时间|	-gc -gcold -gcoldcapacity -gcpermcapacity -gcutil -gccause
GCT|	GC操作花费的总时间。|	-gc -gcold -gcoldcapacity -gcpermcapacity -gcutil -gccause
NGCMN|	新生代最小空间容量，单位KB。|	-gccapacity -gcnewcapacity
NGCMX|	新生代最大空间容量，单位KB。|	-gccapacity -gcnewcapacity
NGC|	新生代当前空间容量，单位KB。|	-gccapacity -gcnewcapacity
OGCMN|	老年代最小空间容量，单位KB。|	-gccapacity -gcoldcapacity
OGCMX|	老年代最大空间容量，单位KB。|	-gccapacity -gcoldcapacity
OGC|	老年代当前空间容量制，单位KB。|	-gccapacity -gcoldcapacity
PGCMN|	持久代最小空间容量，单位KB。|	-gccapacity -gcpermcapacity
PGCMX|	持久代最大空间容量，单位KB。|	-gccapacity -gcpermcapacity  
PGC	|持久代当前空间容量，单位KB。|	-gccapacity -gcpermcapacity
PC|	持久代当前空间大小，单位KB|	-gccapacity -gcpermcapacity
PU|	持久代当前已用空间大小，单位KB|	-gc -gcold
LGCC|	最后一次GC发生的原因|	-gccause
GCC|	当前GC发生的原因|	-gccause
TT|	老年化阈值。被移动到老年代之前，在新生代空存活的次数。|	-gcnew
MTT|	最大老年化阈值。被移动到老年代之前，在新生代空存活的次数。|	-gcnew
DSS|	幸存者区所需空间大小，单位KB。|	-gcnew  

jstat 的好处是它可以持续的监控GC操作数据，不论Java应用是运行在本地还是远程，只要有控制台的地方就可以使用。当使用–gcutil 会输出如下信息。在GC优化的时候，你需要特别注意YGC, YGCT, FGC, FGCT 和GCT。  
```
S0      S1       E        O        P        YGC    YGCT     FGC    FGCT     GCT
0.00    66.44    54.12    10.58    86.63    217    0.928     2     0.067    0.995
0.00    66.44    54.12    10.58    86.63    217    0.928     2     0.067    0.995
0.00    66.44    54.12    10.58    86.63    217    0.928     2     0.067    0.995
```  
这些信息很重要，因为它们展示了GC处理到底花费了多少时间。  
在这个例子中，YGC 是217而YGCT 是0.928，这样在简单的计算数据平均数后，你可以知道每次新生代的GC大概需要4ms（0.004秒），而full GC的平均时间为33ms。  
但是，只看数据平均数经常无法分析出真正的GC问题。这是主要是因为GC操作时间严重的偏差（换句话说，假如两次full GC的时间是 67ms，那么其中的一次full GC可能执行了10ms而另一个可能执行了57ms。）为了更好地检测每次GC处理时间，最好使用 –verbosegc来替代数据平均数。  
**-verbosegc**  
-verbosegc 是在启动一个Java应用时可以指定的JVM参数之一。而jstat 可以监控任何JVM应用，即便它没有制定任何参数。 -verbosegc 需要在启动的时候指定，因此你可能会认为它没有必要（因为jstat可以替代之）。但是， -verbosegc 会以更浅显易懂的方式展现GC发生的结果，因此他对于监控监控GC信息十分有用。  

	|jstat	|-verbosegc
---|---|---
监控对象|	运行在本机的Java应用可以把日志输出到终端上，或者借助jstatd命令通过网络连接远程的Java应用。|	只有那些把-verbogc作为启动参数的JVM。  
输出信息|	堆状态（已用空间，最大限制，GC执行次数/时间，等等）|	执行GC前后新生代和老年代空间大小，GC执行时间。  
输出时间|	Every designated time|每次设定好的时间。	每次GC发生的时候。
何时有用。|	当你试图观察堆空间变化情况|	当你试图了解单次GC产生的效果。  

下面是-verbosegc 的可用参数  
- -XX:+PrintGCDetails  
- -XX:+PrintGCTimeStamps  
- -XX:+PrintHeapAtGC  
- -XX:+PrintGCDateStamps (from JDK 6 update 4)  

如果只是用了 -verbosegc 。那么默认会加上 -XX:+PrintGCDetails。 –verbosgc 的附加参数并不是独立的。而是经常组合起来使用。  
使用 –verbosegc后，每次GC发生你都会看到如下格式的结果。  
```
[GC [<collector>: <starting occupancy1> -> <ending occupancy1>, <pause time1> secs] <starting occupancy3> -> <ending occupancy3>, <pause time3> secs]
```  

收集器|	minor gc使用的收集器的名字。
---|---|---
starting occupancy1|	GC执行前新生代空间大小。
ending occupancy1|	GC执行后新生代空间大小。
pause time1|	因为执行minor GC，Java应用暂停的时间。
starting occupancy3|	GC执行前堆区域总大小
ending occupancy3|	GC执行后堆区域总大小
pause time3|	Java应用由于执行堆空间GC（包括major GC）而停止的时间。  

这是-verbosegc 输出的minor GC的例子。  
```
S0    S1     E      O      P        YGC    YGCT    FGC    FGCT     GCT
0.00  66.44  54.12  10.58  86.63    217    0.928     2    0.067    0.995
0.00  66.44  54.12  10.58  86.63    217    0.928     2    0.067    0.995
0.00  66.44  54.12  10.58  86.63    217    0.928     2    0.067    0.995
```  
这是 Full GC发生时的例子  
```
[Full GC [Tenured: 3485K->4095K(4096K), 0.1745373 secs] 61244K->7418K(63104K), [Perm : 10756K->10756K(12288K)], 0.1762129 secs] [Times: user=0.19 sys=0.00, real=0.19 secs]
```  
如果使用了 CMS collector，那么如下CMS信息也会被输出。  
由于 –verbosegc 参数在每次GC事件发生的时候都会输出日志，我们可以很轻易地观察到GC操作对于堆空间的影响。  
**(Java) VisualVM  + Visual GC**  
Java Visual VM是由Oracle JDK提供的图形化的汇总和监控工具。  
![](http://i.imgur.com/9bWQGaW.png)  
图1: VisualVM 截图  
除了JDK中自带的版本，你还可以直接从官网下载Visual VM。出于便利性的考虑，JDK中包含的版本被命名为Java VisualVM (jvisualvm),而官网提供的版本被命名为Visual VM (visualvm)。两者的功能基本相同，只有一些细小的差别，例如安装组件的时候。就个人而言，我更喜欢可以从官网下载的Visual VM。  
![](http://i.imgur.com/4ETatYk.png)  
图 2: Viusal GC 安装截图  
通过Visual GC，你可以更直观的看到执行jstatd 所得到的信息。  
![](http://i.imgur.com/RYGZ1FH.png)  
图3: Visual GC 执行截图  
**HPJMeter**  
HPJMeter 可以很方便的分析 -verbosegc 输出的结果，如果Visual GC可以视作jstat的图形化版本，那么HPJMeter就相当于 –verbosgc的图形化版本。当然，GC分析只是HPJMeter提供的众多功能之一，HPJMeter是由惠普开发的性能监控工具，他可以支持HP-UX，Linux以及MS Windows。
起初，一个成为HPTune 被设计用来图形化的分析-verbosegc.输出的结果。但是，随着HPTune的功能被集成到HPJMeter 3.0版本之后，就没有必要单独下载HPTune了。但运行一个应用时， -verbosegc 的结果会被输出到一个独立的文件中。  
你可以用HPJMeter直接打开这个文件，以便更直观的分析GC性能数据。  
![](http://i.imgur.com/GcZEVoI.png)  
图4: HPJMeter  
推荐使用jstat 来监控GC操作，如果你感觉到GC操作的执行时间过长，那就可以使用verbosegc 参数来分析GC。GC优化的大体步骤就是在添加verbosegc 参数后，调整GC参数，分析修改后的结果。  
## 优化Java垃圾回收机制  

**为什么需要优化GC**  
或者说的更确切一些，**对于基于Java的服务，是否有必要优化GC？**应该说，对于所有的基于Java的服务，并不总是需要进行GC优化，但前提是所运行的基于Java的系统，包含了如下参数或行为：  
- 已经通过 -Xms 和–Xmx 设置了内存大小  
- 包含了 -server 参数  
- 系统中没有超时日志等错误日志  

**换句话说，如果你没有设定内存的大小，并且系统充斥着大量的超时日志时，你就需要在你的系统中进行GC优化了。**  
但是，你需要时刻铭记一条：**GC优化永远是最后一项任务。**  
想一下进行GC优化的最根本原因，垃圾收集器清除在Java程序中创建的对象，GC执行的次数即需要被垃圾收集器清理的对象个数，与创建对象的数量成正比，因此，首先你**应该减少创建对象的数量。**  
俗话说的好，“冰冻三尺非一日之寒”。我们应该从小事做起，否则日积月累就会很难管理。  
- 我们需要使用StringBuilder 或者StringBuffer 来替代String  
- 应该尽量少的输出日志  

但是，我们知道有些情况会让我们束手无策，我们眼睁睁的看着XML以及JSON解析占用了大量的内存。即便我们已经尽可能少的使用String以及尽量少的输出日志，大量的临时内存被用于XML或者JSON解析，例如10-100MB。但是，舍弃XML和JSON是很难的。我们只要知道，他会占用很多内存。  
如果应用内存使用量经过几次重复调整之后有所改善，你就可以开始GC优化了  
我为GC优化归纳了两个目的：  
1.一个是将转移到老年代的对象数量降到最少  
2.另一个是减少Full GC的执行时间  
**将转移到老年代的对象数量降到最少**  
按代的GC机制由Oracle JVM提供，不包括可以在JDK7以及更高版本中使用的G1 GC。换句话说，对象被创建在伊甸园空间，而后转化到幸存者空间，最终剩余的对象被送到老年代。某些比较大的对象会在被创建在伊甸园空间后，直接转移到老年代空间。老年代空间上的GC处理会比新生代花费更多的时间。因此，减少被移到老年代对象的数据可以显著地减少Full GC的频率。减少被移到老年代空间的对象的数量，可能被误解为将对象留在新生代。但是，这是不可能的。取而代之，你可以**调整新生代空间的大小。**  
**减少Full GC执行时间**  
Full GC的执行时间比Minor GC要长很多。因此，如果Full GC花费了太多的时间（超过1秒），一些连接的部分可能会发生超时错误。  
- 如果你试图通过消减老年代空间来减少Full GC的执行时间，可能会导致OutOfMemoryError 或者 Full GC执行的次数会增加。  
- 与之相反，如果你试图通过增加老年代空间来减少Full GC执行次数，执行时间会增加。  

因此，你需要**将老年代空间设定为一个“合适”的值。**  
**影响GC性能的参数**  
**不要幻想“某个人设定了GC参数后性能得到极大的提高，我们为什么不和他用一样的参数？”**，因为不同的Web服务所创建对象的大小和他们的生命周期都不尽相同。  
简单来说，如果一个任务的执行条件是A，B，C，D和E，同样的任务执行条件换为A和B，你会觉得哪个更快？从一般人的直觉来看，在A和B条件下执行的任务会更快。  
Java GC参数也是相同的道理，设定一些参数不但没有提高GC执行速度，反而可能导致他更慢。**GC优化的最基本原则**是将不同的GC参数用于2台或者多台服务器，并进行对比，并将那些被证明提高了性能或者减少了GC执行时间的参数应用于服务器。请谨记这一点。  
下面这个表格列出了GC参数中与内存大小相关的，可以影响性能的参数。  
表1：GC优化需要考虑的Java参数  

定义|参数|描述
---|---|---
堆内存空间|-Xms|Heap area size when starting JVM<br/>启动JVM时的堆内存空间。
 |-Xmx|Maximum heap area size 堆内存最大限制  
新生代空间|-XX:NewRatio|Ratio of New area and Old area<br/>新生代和老年代的占比   
 |-XX:NewSize|New area size<br/>新生代空间
 |-XX:SurvivorRatio|Ratio ofEdenarea and Survivor area<br/>伊甸园空间和幸存者空间的占比

我在进行GC优化时经常使用-Xms，-Xmx和-XX:NewRatio。-Xms和-Xmx是必须的。你如何设定NewRatio 会对GC性能产生十分显著的影响。有些人可能会问如何设定**Perm**区域的大小？你可以通过-XX:PermSize 和-XX:MaxPermSize参数来设定  
当OutOfMemoryError 错误发生并且是由于Perm空间不足导致时，另一个可能影响GC性能的参数是GC类型。下表列出了所有可选的GC类型（基于JDK6.0）  
表2：GC类型可选参数  

分类|参数|备考
---|---|---
Serial GC|-XX:+UseSerialGC| 
Parallel GC|-XX:+UseParallelGC<br/>-XX:ParallelGCThreads=value| 
Parallel Compacting GC|-XX:+UseParallelOldGC| 
CMS GC|-XX:+UseConcMarkSweepGC<br/>-XX:+UseParNewGC<br/>-XX:+CMSParallelRemarkEnabled<br/>-XX:CMSInitiatingOccupancyFraction=value<br/>-XX:+UseCMSInitiatingOccupancyOnly|   
G1|-XX:+UnlockExperimentalVMOptions<br/>-XX:+UseG1GC|在JDK6中这两个参数必须同时使用  

除了G1 GC，可以通过每种类型第一行的参数来切换GC类型。最常用的GC类型是Serial GC。他专门针对客户端系统进行了优化。  
影响GC性能的参数有很多，但是上面提到的参数会带来最显著的效果。请牢记，设定过多的参数不一定会减少GC执行时间。  
**GC优化过程**  
GC优化的过程与大多数性能改善的过程极其类似。下面是我使用的GC优化过程。  
**1.监控GC状态**  
首先你需要监控GC来检查在系统执行过程中GC的各种状态。  
**2.在分析监控结果后，决定是否进行GC优化**  
在检查GC状态的过程中，你应该分析监控结果以便决定是否进行GC优化，如果分析结果表明执行GC的时间只有0.1-0.3秒，那你就没必要浪费时间去进行GC优化。但是，如果GC的执行时间是1-3秒，或者超过10秒，GC将势在必行。  
但是，如果你已经为Java分配了10GB的内存，并且不能再减少内存大小，你将无法再对GC进行优化。在进行GC优化之前，你必须想清楚你为什么要分配如此大的内存空间。假如当你分1 GB 或 2 GB内存时出现OutOfMemoryError ，你应该执行堆内存转储（heap dump），并消除隐患。  
**注意：**  
堆内存转储是一个用来检查Java内存中的对象和数据的文件。该文件可以通过执行JDK中的jmap命令来创建。在创建文件的过程中，Java程序会暂停，因此不要再系统执行过程中创建该文件。  
**3. 调整GC类型/内存空间**  
如果你已经决定要进行GC优化，那么就要选择GC类型和设定内存空间。在这时，如果你有几台不同服务器，请时刻牢记，检查每一台服务器的GC参数，并进行有针对性的优化。  
**4.分析结果**  
在调整了GC参数并持续收集24小时之后，开始对结果进行分析，如果你幸运的话，你就找到那些最适合系统的GC参数。反之，你需要通过分析日志来检查内存是如何被分配的。然后你需要通过不断的调整GC类型和内存空间大小一边找到最佳的参数。  
**5. 如果结果令人满意，你可以将该参数应用于所有的服务器，并停止GC优化**  
有过GC优化结果令人满意，你可以应用于所有的服务器，下面的章节中，我们将看到每个步骤的具体任务。  
**监控GC状态及分析结果**  
查看运行中的Web Application Server (WAS)的GC状态的最佳方法是通过jstat命令  
下面这个例子展现了某个JVM在进行GC优化之前的状态。  
（很遗憾，这不是一个操作服务器）  
```
$ jstat -gcutil 21719 1s
S0    S1    E    O    P    YGC    YGCT    FGC    FGCT GCT
48.66 0.00 48.10 49.70 77.45 3428 172.623 3 59.050 231.673
48.66 0.00 48.10 49.70 77.45 3428 172.623 3 59.050 231.673
```  
如上表，我们先看一下YGC 和YGCT，计算YGCT/ YGC得到0.050秒（50毫秒）。这意味着新生代空间上的GC操作平均花费50毫秒。在这种情况，你大可不必担心新生代空间上执行的GC操作。
接下来，我们来看一下FGCT 和FGC。，计算FGCT/ FGC得到19.68秒，这意味着GC的平均执行时间为19.68秒，可能是每次花费19.68秒执行了三次，也可能是其中的两次执行了1秒而另一次执行了58秒。不论哪种情况，都需要进行GC优化。  
通过**jstat** 命令可以很轻易地查看GC状态，但是，分析GC的最佳方式是通过–verbosegc参数来生成日志，在之前的文章中我已经解释了如何分析这些日志，**HPJMeter** 是我个人最喜欢的用于分析-verbosegc 日志的工具。他很易于使用和分析结果。通过HPJmeter你可以很轻易查看GC执行时间以及GC发生频率。如果GC执行时间满足下面所有的条件，就意味着无需进行GC优化了。  
- Minor GC执行的很快（小于50ms）  
- Minor GC执行的并不频繁（大概10秒一次）  
- Full GC执行的很快（小于1s）  
- Full GC执行的并不频繁（10分钟一次）  

上面提到的数字并不是绝对的；他们根据服务状态的不同而有所区别，某些服务可能满足于Full GC每次0.9秒的速度，但另一些可能不是。因此，针对不同的服务设定不同的值以决定是否进行GC优化。  
在查看GC状态的时候有件事你需要特别注意，那就是不要只关注Minor GC 和Full GC的执行时间。还要关注**GC执行的次数**,例如，当新生代空间较小时，Minor GC会过于频繁的执行（有时每秒超过1次）。另外，转移到老年代的对象数增多，则会导致Full GC执行次数增多。因此，别忘了加上–gccapacity参数来查看具体占用了多少空间。  
**设定GC类型/内存空间大小**  
- **设定GC类型**  

OracleJVM有5种GC类型，但是在JDK7之前的版本中，只能在Parallel GC, Parallel Compacting GC 和CMS GC之中选择一个，对于选择哪个没有明确的原则和规则。  
这样的话，**我们该如何选择呢？**强烈建议三者都选，但是，有一点是很明确的：CMS GC比Parallel GCs更快。如果真的如此，那么就选CMS GC了。但是，CMS GC也不总是更快。整体来看，CMS GC模式下的Full GC执行更快，不过，一旦出现并行模式失败，他将比Parallel GC更慢。  
**并发模式失败**  
我们来详细讲解一下并发模式失败。  
Parallel GC 和 CMS GC 最大的不同来自于压缩任务。压缩任务是通过删除已分配内存空间中的空白空间以便压缩内存，清理内存碎片。  
在Parallel GC模式下，压缩工作在Full GC执行时进行，这会费很多时间，但是，在执行完Full GC之后，由于能够顺序地分配空间，随后的内存能够被更快的分配。  
与之相反的，CMS GC并不进行压缩处理，因此，CMS GC执行的更快。但是，由于没有压缩，在进行磁盘清理之前，内存中会有很多空白空间。这就是说，可能没有足够的空间存储大的对象，例如，虽然老年代空间还有300MB空间，但是一些10MB的对象无法被顺序的存储。在这种情况下，会出现“并行模式失败”警告，并执行压缩处理。在CMS GC模式下，压缩处理的执行时间要比Parallel GCs长很多。另外，这还将导致另外一个问题。关于并发模式失败的详细说明，可以参考Oracle工程师撰写的[Understanding CMS GC Logs](https://blogs.oracle.com/poonam/entry/understanding_cms_gc_logs)。  
综上所述，你需要找到最适合你的系统的GC类型。  
每个系统都有最适合他的GC类型等着你去寻找，如果你有6台服务器。我建议你每两台设置相同的参数。并添加 –verbosegc参数，分析结果。  
- **设定内存空间大小**  

下表展示了内存空间大小，GC执行次数以及GC执行时间三者间的关系。  
  - 大内存空间  
    - 减小GC执行次数  
    - 增加GC执行时间  
  - 小内存空间  
    - 减小GC执行时间  
    - 增加GC执行次数  

关于如何设置内存空间的大小，没有唯一的标准答案。如果服务器资源足够，而且Full GC也可能在1秒内完成，设置为10GB当然可行。。但绝大多数服务器并不是这样，当内存设为10GB时，可能要花费10~30秒来执行Full GC。当然，执行时间会随对象的大小而改变。  
鉴于如此，我们应该如何设定内存空间大小呢？一般来说，我建议为500MB。不过请注意这不是让你将WAS的内存参数设置为–Xms500m 和–Xmx500m。根据优化GC之前的状态，如果Full GC执行之后内存空间剩余300MB，那么最好将内存设置为1GB（300MB（默认程序占用）+ 500MB（老年代最小空间）+200MB（空闲内存））。也就是说你要为老年代额外设置500MB。因此，如果你有三个执行服务器，内存分别设置为1GB，1.5GB，2GB，并且检查结果。  
理论上来讲，GC执行速度应该遵循1GB> 1.5GB> 2GB,因此1GB执行GC速度最快。但是并不说明1GB空间的Full GC会花费1秒而2GB空间会花费2秒。时间取决于服务器的性能和对象的大小。因此，最佳的方式是建立尽可能多的衡量指标来监控他们。  
对于内存空间大小，你应该额外设定NewRatio参数。NewRatio参数是新生代和老年代空间的比例，即XX:NewRatio=1意味着新生代与老年代之比为1:1。对于1GB来说就是新生代和老年代各500MB。如果NewRatio为2，意味着新生代老年代之比为1:2，因此该值越大，老年代空间越大，新生代空间越小。  
这看似一件不是很重要的事情，但NewRatio参数会显著地影响整个GC的性能。如果新生代空间很小，会用更多的对象被转移到老年代空间，这样导致频繁的Full GC，增加暂停时间。  
你可以简单的认为NewRatio 为1是最佳的选择，但是，有时可能设置为2或3更好，我就见过很多这样的例子。  
如何最快的完成GC优化？对比性能测试的结果应该是最快地方法，为每一台服务器设置不同的参数并监控他们的状态，强烈建议至少监控1或2天的数据。但是，当你对GC优化是，你要确保每次执行相同的负载。并且请求的比率，例如URL都应该是一致的。不过，即便对于专业测试人员要想精确的控制负载也是很难的，并要花费大量的时间准备。因此，相对来说比较方便和容易的方法是调整参数，之后花费较长的时间收集结果。  
**分析GC优化结果**  
在设置了GC参数以及-verbosegc参数之后，通过tail命令确保日志被正确的生成。如果参数设置的不正确或者日志没有生成，你将白白浪费你的时间。如果日志正确的话，持续收集1到2天。随后最好将日志下载到本地PC并用**HPJMeter**来分析  
- Full GC 执行时间  
- Minor GC执行时间  
- Full GC 执行间隔  
- Minor GC 执行间隔  
- Entire Full GC 执行时间  
- Entire Minor GC 执行时间  
- Entire GC 执行时间  
- Full GC e执行时间  
- Minor GC 执行时间  

找到最佳的GC参数是件非常幸运的事情，然而在大多数场合，我们并不会得到幸运之神的眷顾，在进行GC优化时要尽量小心谨慎，想一步完成优化往往会导致OutOfMemoryError 。  
**优化示例**  
好了，我们一直在纸上谈兵，现在我们看一些实际的GC优化的例子。  
	**示例1**  
	下面这个例子针对 **Service S**的优化,对于最近被部署的 Service S，Full GC花费了太长的时间。  
	请看 jstat –gcutil的执行结果。  
	```
	S0 S1 E O P YGC YGCT FGC FGCT GCT
	12.16 0.00 5.18 63.78 20.32 54 2.047 5 6.946 8.993
	```  
最左边的Perm 空间对于最初的GC优化不是很重要，这一次YGC参数的值更加有用。  
Minor GC和Full GC的平均值如下表所示  
表3：Service S的Minor GC 和Full GC的平均执行时间  

GC 类型|GC 执行次数|GC 执行时间|平均
---|---|---|---
Minor GC | 54 | 2.047 | 37 ms
Full GC | 5 | 6.946 | 1,389 s  

最重要的是下面两个数据  
- 新生代实际使用空间: 212,992 KB  
- 老年代实际使用空间: 1,884,160 KB  

因此，总的内存空间为2GB,不算Perm空间的话，新生代与老年代之比为1:9。通过jstat和-verbosegc 日志进行数据收集，并把三台服务器按照如下方式设置。  
- NewRatio=2  
- NewRatio=3  
- NewRatio=4  

一天之后，检查系统的GC日志后发现，在设置了NewRatio参数后很幸运的没有发生Full GC，  
**为什么？**  
- NewRatio=2: 45 ms  
- NewRatio=3: 34 ms  
- NewRatio=4: 30 ms  

我们看到NewRatio=4 是最佳的参数，虽然它的新生代空间最小，但GC时间确最短。设定这个参数之后，系统没有执行过Full GC。  
为了说明这个问题，下面是服务之星一段时间后执行jstat –gcutil的结果  
```
S0 S1 E O P YGC YGCT FGC FGCT GCT
8.61 0.00 30.67 24.62 22.38 2424 30.219 0 0.000 30.219
```  
你可能会认为因为服务器接受的请求少才导致的GC执行频率下降。实际上，虽然Full GC没有执行，但是Minor GC被执行了 2,424次。  
**示例2**  
这是一个针对ServiceA的例子，我们通过公司内部的应用性能管理系统（APM）发现JVM暂停了相当长的时间（超过8秒），因此我们进行了GC优化。我们找到了Full GC执行时间过长的原因，并着手解决。  
进行GC优化的第一步，就是我们添加了-verbosegc参数，并得到如下结果。  
![](http://i.imgur.com/nvwOPVs.jpg)  
图1：进行GC优化之前的STW时间  
如上图所示，由HPJMeter自动生成的图片之一。X坐标表示JVM执行的时间。Y坐标表示每次GC的时间。CMS绿点，表示Full GC结果。Parallel Scavenge蓝点，表示Minor GC结果。  
之前我曾经说过CMS GC是最快的，但是上面的的结果显示出于某种原因，它最多花费了15秒。是**什么导致这个结果？**是否想起我之前提过的，CMS在进行内存清理时，会变慢。与此同时，服务的内存被设定为 –Xms1g和–Xmx4g ，且实际分配了4GB内存。  
因此，我将GC类型从CMS改为Parallel GC。并且将内存改为2GB，设定NewRatio 为3。几小时之后我使用 jstat –gcutil得到如下结果  
```
S0 S1 E O P YGC YGCT FGC FGCT GCT
0.00 30.48 3.31 26.54 37.01 226 11.131 4 11.758 22.890
```  
相对于4GB时的15秒，Full GC变成了平均每次3秒。但是3秒一样比较慢，因此我设计了如下6种场景。  
- Case 1: -XX:+UseParallelGC -Xms1536m -Xmx1536m -XX:NewRatio=2  
- Case 2: -XX:+UseParallelGC -Xms1536m -Xmx1536m -XX:NewRatio=3  
- Case 3: -XX:+UseParallelGC -Xms1g -Xmx1g -XX:NewRatio=3  
- Case 4: -XX:+UseParallelOldGC -Xms1536m -Xmx1536m -XX:NewRatio=2  
- Case 5: -XX:+UseParallelOldGC -Xms1536m -Xmx1536m -XX:NewRatio=3  
- Case 6: -XX:+UseParallelOldGC -Xms1g -Xmx1g -XX:NewRatio=3  

**哪一个最快呢？**结果显示，内存越小，结果越好。下图展示了Case6的结果。这是GC的性能最好。最长的响应时间只有1.7秒。平均时间在1秒之内。  
![](http://i.imgur.com/QtKgkWn.png)  
图2：Case6的时间图表  
基于以上结果。我们按照Case6调整了GC参数。但是，这导致了每天晚上都会发生OutOfMemoryError。在这里很难解释具体的原因。简单来说，批处理程序导致了内存泄漏。相关的问题已经被解决。  
如果对GC日志只分析很短的时间就贸然对所有服务器进行优化是非常危险的。请时刻牢记，你必须同时分析GC日志和应用程序。  
我们回顾了两个关于GC优化的例子，正如我之前提到的，例子中提到的GC参数，可以设置在相同的服务器之上，但前提是他们具有相同的CPU，操作系统，JDK版本以及运行着相同的服务。但是不要直接把我用过的参数用到你的服务至上，它们未必能很好的工作。  
**结论**  
我凭借经验进行GC优化，而没有执行堆转储并分析内存的详细内容。精确地分析内存可以得到更好的优化效果。但是，这种分析一般适用于内存使用量相对固定的场合。不过，如果服务严重过载并占用的大量的内存，强力建议根据之前的经验进行GC优化。  
我已经在一些服务上设置了G1 GC参数，并进行过性能测试。但还没有应用与正式环境，G1 GC参数的速度要快于其他任何GC类型。但是，你必须要升级到JDK7。另外，他的稳定性也暂时没有保障，没人知道是否会出现致命的错误。因此还不到将其正式应用的时候  
在未来的某一天，等到JDK7真正稳定了（这不是说他现在不稳定），并且WAS针对JDK7进行优化后，G1 GC最终能够按照预期的那样工作了，我们可能就不需要在进行GC优化了。  

## Apache的MaxClients参数详解及其在Tomcat执行FullGC时的影响  
将阐述Apache中MaxClients 参数的重要性，以及他如何在GC发生时，显著地影响整个系统的性能。我将提供几个例子以方便你理解MaxClients 导致的问题。同时我还会说明如何根据系统的内存情况，设置最佳的MaxClients参数值。   
**MaxClients对于系统的影响**  
大部分开发人员都知道在由于GC发生而导致的”停止世界现象(STW) “（详细请参见[Understanding Java Garbage Collection](http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/)）。由于[Java 虚拟机](http://www.cubrid.org/blog/dev-platform/understanding-jvm-internals/) (JVM)管理着内存，以Java为基础的程序无法摆脱GC导致的STW现象。假如在某一个时间，当你正在操作你开发的应用时，GC开始执行。即使TTS错误没有发生，你的服务也会给客户展现未预期的503错误。  
**服务执行环境**  
由于架构本身的特点，相比较而言纵向扩展，Web服务更适合横向扩展（译者注:增加服务器的数量，而不是提高件配置）。因此，总体来讲，物理设备会根据性能要求被配置成1台Apache+n台Tomcat。但是本文假设我们的环境是1台Apache+一台Tomcat同时安装在一台主机行，如下图所示。  
![](http://i.imgur.com/EonO35v.png)  
图1：本文假射的服务执行环境  
仅供参考，本文描述的参数基于Apache 2.2.21 (prefork MPM)，Tomcat 6.0.35，CentOS 4.72 (32-bit)，jdk 1.6.0_24。  
系统可用内存2GB，垃圾收集器使用ParallelOldGC，AdaptiveSizePolicy采用默认的设置true，堆内存空间600M  
**STW 和HTTP 503**  
让我们假设访问Apache的请求为 200 req/s且有10个httpd进程在运行，另外我们暂时不考虑每个请求的响应时间。在这种前提下，我们假设由于full GC导致的暂停时间为1秒。**当Full GC发生的时候Tomcat会怎样？**  
第一件进入你脑海的事情应该是Tomcat会因为full GC而停止响应任何请求。在这种情况下，**当Tomcat暂停响应请求时Apache会发生什么？**  
当Tomcat暂停时，请求会以200 req/s的速度不断的涌入Apache。一般来说，在Full GC发生之前，请求的响应可以快速地被10个或更多的httpd进程处理掉。但是，因为Tomcat暂停了，httpd进程会被不停地创建以相应新进请求。直到超过**httpd.conf** 文件中定义 MaxClients 为止。由于默认值为256，Apache不会在乎请求以200 req/s的速度涌入。  
这时，**新创建的httpd线程将如何呢？**  
Httpd进程通过**mod_jk** 模块所管理的空闲的AJP连接，将请求转发给Tomcat。如果没有空闲连接，他会申请创建新的连接。但是，因为Tomcat暂停了，创建新连接的请求会被拒绝。因此这些请求会被存储在backlog队列中，数量的多少取决于**server.xml**中关于AJP Connector的设置。一旦请求数量超过backlog队列的空间限制。Apache就会返回拒绝连接错误。并且返回**HTTP 503** 错误给用户。  
在这种假设条件下，默认的backlog队列空间是100，而请求到达速度是200 req/s。因此，full GC导致的一秒钟的暂停会使得超过100个请求返回503错误。  
在这种假设条件下，默认的backlog队列空间是100，而请求到达速度是200 req/s。因此，full GC导致的一秒钟的暂停会使得超过100个请求返回503错误。  
这样，当Full GC结束后，backlog队列中存储的内容会被Tomcat接受并在通过工作线程处理，线程的最大数量取决于MaxThreads的值（默认200）。  
**MaxClients 与backlog**  
在这种情况下，**设定哪个参数可以避免返回给用户503错误呢？**  
首先，我们应该知道backlog的值要够大，以至于能够容纳所有因为Full GC导致暂停期间涌入的请求。换句话说应该不小于200。  
那么，**这么设置之后会不会产生新的问题呢？**  
让我们假设将backlog设置为200后再重复一下上面的过程。得到的结果比之前更加严重。系统内存使用量一般情况下为50%，但是，在发生Full GC时快速增加到100%，同时导致交换内存空间快速增加，更为严重的是导致Full GC的暂停时间从1秒变成了4秒甚至更多，系统在此期间完全宕机，不能响应任何请求。  
在第一种情况下，只有100或更多的请求返回503错误。但是，当我们把backlog调整到200后，超过500个请求会挂起3秒甚至更多地时间无法得到应答  
上面这个例子可以很好的说明当你没有完全理解各个设置之间的内在关系时（例如，对于系统的影响），盲目修改系统会导致什么后果。  
那么，**为什么会产生这个现象呢？**  
问题的根源在于 MaxClients 参数的特性。  
将MaxClients 设置为一个很大的值本身没有问题，但最重要的是在设定MaxClients 参数时，你要确保即使等同于MaxClients 数量的httpd进程被同时创建，**内存使用量也不会超过80%**。  
系统的内存交换参数一般被设定为60（默认）。因此，当内存使用量超过80%时，就会进行内存交换。  
让我们再来看一下为什么这个特性会导致上面那个严重的问题。当请求以200 req/s的速度涌向Tomcat时，Tomcat由于full GC暂停了。此时backlog被设置为200。Apache大约创建100个httpd进程。在这种情况下，一旦内存使用量超过80%，操作系统会激活交换内存区域，并且由于系统认为JVM的老年代中的对象在很长一段时间内未被使用，而将他们移动到交换区域。  
最终的结果是，GC使用了内存交换空间，暂停时间剧增。因此httpd进程数进一步增加。从而导致上面描述的内存使用量达到100%的情况。  
这两个场合的唯一区别就是backlog的值：100 vs.200。**为什么只在200的情况下发生？**

两者不同的原因在于创建的httpd进程的数量。当backlog设置为100时并且Full GC发生时，会创建100个请求的连接并保存在backlog队列中。其他请求得到拒绝连接错误信息并发挥503错误。因此，总的httpd 进程数量仅仅会略高于100。而当backlog被设置为200时，200个请求会创建连接，因此。总的httpd进程数会多于200。这样超过阀值，从而导致内存交换的发生。紧接着，不考虑内存使用量而的设定 MaxClients参数，Full GC导致httpd进程数量暴增，引发内存交换，降低系统性能。  
**MaxClients参数的计算公式**  
如果系统的内存使2GB，MaxClients 的值在任何情况下都不应该超过内存的80%（1.6GB），以避免由于内存交换导致的性能下降。换句话说。1.6GB的内存应该共享和分配给Apache，Tomcat以及那些默认被安装的代理程序。  
让我们假设代理程序被默认安装在系统，并占用了200m内存，对于Tomcat堆内存的-Xmx 被设定为 600m。因此根据top命令的结果，Tomcat会一直占用725m（Perm Gen + Native Heap Area）。最终Apache可以使用700m内存空间。如下所示。  
![](http://i.imgur.com/rwBVo27.png)  
图2：测试系统的top截屏  
如上所述，**我们将内存设为700m后MaxClients 应该是多少呢？**  
这要取决于加载模块的数量，对于NHN Web服务来说。Apache只是个简单的代理转发，每个httpd线程4m内存（根据top命令的结果）足以（参见图2）。因此。700m内存对应的 MaxClients应该是175。  
**总结**  
一个健壮的服务配置至少应该能够降低在服务过载时宕机的时间，在合理的范围内成功的应答请求。针对基于Java的Web服务。你必须检查你的服务在Full GC导致的STW时间内能否稳定的响应请求。  
为了响应更多的用户请求和应对DDoS攻击，在没有全面考虑系统内存等因素的情况下，贸然地将 MaxClients设置为一个很大的值，那么它将失去作为阀值的功能，而导致系统出现更严重的问题。  

  

   

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





  

   

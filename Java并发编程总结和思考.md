**为什么需要并发**  
并发其实是一种解耦合的策略，它帮助我们把做什么（目标）和什么时候做（时机）分开。这样做可以明显改进应用程序的吞吐量（获得更多的CPU调度时间）和结构（程序有多个部分在协同工作）。做过Java Web开发的人都知道，Java Web中的Servlet程序在Servlet容器的支持下采用单实例多线程的工作模式，Servlet容器为你处理了并发问题。  
**误解和正解**  
最常见的对并发编程的误解有以下这些：  
-并发总能改进性能（并发在CPU有很多空闲时间时能明显改进程序的性能，但当线程数量较多的时候，线程间频繁的调度切换反而会让系统的性能下降）  
-编写并发程序无需修改原有的设计（目的与时机的解耦往往会对系统结构产生巨大的影响）  
-在使用Web或EJB容器时不用关注并发问题（只有了解了容器在做什么，才能更好的使用容器）  
下面的这些说法才是对并发客观的认识：  
-编写并发程序会在代码上增加额外的开销  
-正确的并发是非常复杂的，即使对于很简单的问题  
-并发中的缺陷因为不易重现也不容易被发现  
-并发往往需要对设计策略从根本上进行修改
**并发编程的原则和技巧**  
**单一职责原则**  
分离并发相关代码和其他代码（并发相关代码有自己的开发、修改和调优生命周期）。  
**限制数据作用域**  
两个线程修改共享对象的同一字段时可能会相互干扰，导致不可预期的行为，解决方案之一是构造临界区，但是必须限制临界区的数量。  
**使用数据副本**  
数据副本是避免共享数据的好方法，复制出来的对象只是以只读的方式对待。Java 5的java.util.concurrent包中增加一个名为CopyOnWriteArrayList的类，它是List接口的子类型，所以你可以认为它是ArrayList的线程安全的版本，它使用了写时复制的方式创建数据副本进行操作来避免对共享数据并发访问而引发的问题。  
**线程应尽可能独立**  
让线程存在于自己的世界中，不与其他线程共享数据。有过Java Web开发经验的人都知道，Servlet就是以单实例多线程的方式工作，和每个请求相关的数据都是通过Servlet子类的service方法（或者是doGet或doPost方法）的参数传入的。只要Servlet中的代码只使用局部变量，Servlet就不会导致同步问题。springMVC的控制器也是这么做的，从请求中获得的对象都是以方法的参数传入而不是作为类的成员，很明显Struts 2的做法就正好相反，因此Struts 2中作为控制器的Action类都是每个请求对应一个实例。  

**Java 5以前的并发编程**  
Java的线程模型建立在抢占式线程调度的基础上，也就是说：  
- 所有线程可以很容易的共享同一进程中的对象。  
- 能够引用这些对象的任何线程都可以修改这些对象。  
- 为了保护数据，对象可以被锁住。  

Java基于线程和锁的并发过于底层，而且使用锁很多时候都是很万恶的，因为它相当于让所有的并发都变成了排队等待。  
在Java 5以前，可以用synchronized关键字来实现锁的功能，它可以用在代码块和方法上，表示在执行整个代码块或方法之前线程必须取得合适的锁。对于类的非静态方法（成员方法）而言，这意味这要取得对象实例的锁，对于类的静态方法（类方法）而言，要取得类的Class对象的锁，对于同步代码块，程序员可以指定要取得的是那个对象的锁。  
不管是同步代码块还是同步方法，每次只有一个线程可以进入，如果其他线程试图进入（不管是同一同步块还是不同的同步块），JVM会将它们挂起（放入到等锁池中）。这种结构在并发理论中称为临界区（critical section）。这里我们可以对Java中用synchronized实现同步和锁的功能做一个总结：  
- 只能锁定对象，不能锁定基本数据类型  
- 被锁定的对象数组中的单个对象不会被锁定  
- 同步方法可以视为包含整个方法的synchronized(this) { … }代码块  
- 静态同步方法会锁定它的Class对象  
- 内部类的同步是独立于外部类的  
- synchronized修饰符并不是方法签名的组成部分，所以不能出现在接口的方法声明中  
- 非同步的方法不关心锁的状态，它们在同步方法运行时仍然可以得以运行  
- synchronized实现的锁是可重入的锁。  

在JVM内部，为了提高效率，同时运行的每个线程都会有它正在处理的数据的缓存副本，当我们使用synchronzied进行同步的时候，真正被同步的是在不同线程中表示被锁定对象的内存块（副本数据会保持和主内存的同步，现在知道为什么要用同步这个词汇了吧），简单的说就是在同步块或同步方法执行完后，对被锁定的对象做的任何修改要在释放锁之前写回到主内存中；在进入同步块得到锁之后，被锁定对象的数据是从主内存中读出来的，持有锁的线程的数据副本一定和主内存中的数据视图是同步的 。  
在Java最初的版本中，就有一个叫volatile的关键字，它是一种简单的同步的处理机制，因为被volatile修饰的变量遵循以下规则：  
- 变量的值在使用之前总会从主内存中再读取出来。  
- 对变量值的修改总会在完成之后写回到主内存中。  

使用volatile关键字可以在多线程环境下预防编译器不正确的优化假设（编译器可能会将在一个线程中值不会发生改变的变量优化成常量），但只有修改时不依赖当前状态（读取时的值）的变量才应该声明为volatile变量。  
不变模式也是并发编程时可以考虑的一种设计。让对象的状态是不变的，如果希望修改对象的状态，就会创建对象的副本并将改变写入副本而不改变原来的对象，这样就不会出现状态不一致的情况，因此不变对象是线程安全的。Java中我们使用频率极高的String类就采用了这样的设计。如果对不变模式不熟悉，可以阅读阎宏博士的《Java与模式》一书的第34章。说到这里你可能也体会到final关键字的重要意义了。  
**Java 5的并发编程**  
Java 5主要包括以下内容：  
- 更好的线程安全的容器  
- 线程池和相关的工具类  
- 可选的非阻塞解决方案  
- 显示的锁和信号量机制  

下面我们对这些东西进行一一解读。  
**原子类**  
Java 5中的java.util.concurrent包下面有一个atomic子包，其中有几个以Atomic打头的类，例如AtomicInteger和AtomicLong。它们利用了现代处理器的特性，可以用非阻塞的方式完成原子操作，代码如下所示：  
```java
/**
 ID序列生成器
*/
public class IdGenerator {
    private final AtomicLong sequenceNumber = new AtomicLong(0);
 
    public long next() {
        return sequenceNumber.getAndIncrement(); 
    }
}
```  
**显示锁**  
基于synchronized关键字的锁机制有以下问题：  
- 锁只有一种类型，而且对所有同步操作都是一样的作用  
- 锁只能在代码块或方法开始的地方获得，在结束的地方释放  
- 线程要么得到锁，要么阻塞，没有其他的可能性  

Java 5对锁机制进行了重构，提供了显示的锁，这样可以在以下几个方面提升锁机制：  
- 可以添加不同类型的锁，例如读取锁和写入锁  
- 可以在一个方法中加锁，在另一个方法中解锁  
- 可以使用tryLock方式尝试获得锁，如果得不到锁可以等待、回退或者干点别的事情，当然也可以在超时之后放弃操作  

显示的锁都实现了java.util.concurrent.Lock接口，主要有两个实现类：  
- ReentrantLock – 比synchronized稍微灵活一些的重入锁  
- ReentrantReadWriteLock – 在读操作很多写操作很少时性能更好的一种重入锁  

只有一点需要提醒，解锁的方法unlock的调用最好能够在finally块中，因为这里是释放外部资源最好的地方，当然也是释放锁的最佳位置，因为不管正常异常可能都要释放掉锁来给其他线程以运行的机会。  
**CountDownLatch**  
CountDownLatch是一种简单的同步模式，它让一个线程可以等待一个或多个线程完成它们的工作从而避免对临界资源并发访问所引发的各种问题。下面借用别人的一段代码（我对它做了一些重构）来演示CountDownLatch是如何工作的。  
```java
import java.util.concurrent.CountDownLatch;
 
/**
 * 工人类
 * @author 骆昊
 *
 */
class Worker {
    private String name;        // 名字
    private long workDuration;  // 工作持续时间
 
    /**
     * 构造器
     */
    public Worker(String name, long workDuration) {
        this.name = name;
        this.workDuration = workDuration;
    }
 
    /**
     * 完成工作
     */
    public void doWork() {
        System.out.println(name + " begins to work...");
        try {
            Thread.sleep(workDuration); // 用休眠模拟工作执行的时间
        } catch(InterruptedException ex) {
            ex.printStackTrace();
        }
        System.out.println(name + " has finished the job...");
    }
}
 
/**
 * 测试线程
 * @author 骆昊
 *
 */
class WorkerTestThread implements Runnable {
    private Worker worker;
    private CountDownLatch cdLatch;
 
    public WorkerTestThread(Worker worker, CountDownLatch cdLatch) {
        this.worker = worker;
        this.cdLatch = cdLatch;
    }
 
    @Override
    public void run() {
        worker.doWork();        // 让工人开始工作
        cdLatch.countDown();    // 工作完成后倒计时次数减1
    }
}
 
class CountDownLatchTest {
 
    private static final int MAX_WORK_DURATION = 5000;  // 最大工作时间
    private static final int MIN_WORK_DURATION = 1000;  // 最小工作时间
 
    // 产生随机的工作时间
    private static long getRandomWorkDuration(long min, long max) {
        return (long) (Math.random() * (max - min) + min);
    }
 
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(2);   // 创建倒计时闩并指定倒计时次数为2
        Worker w1 = new Worker("骆昊", getRandomWorkDuration(MIN_WORK_DURATION, MAX_WORK_DURATION));
        Worker w2 = new Worker("王大锤", getRandomWorkDuration(MIN_WORK_DURATION, MAX_WORK_DURATION));
 
        new Thread(new WorkerTestThread(w1, latch)).start();
        new Thread(new WorkerTestThread(w2, latch)).start();
 
        try {
            latch.await();  // 等待倒计时闩减到0
            System.out.println("All jobs have been finished!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```  
**ConcurrentHashMap**  
ConcurrentHashMap是HashMap在并发环境下的版本，大家可能要问，既然已经可以通过Collections.synchronizedMap获得线程安全的映射型容器，为什么还需要ConcurrentHashMap呢？因为通过Collections工具类获得的线程安全的HashMap会在读写数据时对整个容器对象上锁，这样其他使用该容器的线程无论如何也无法再获得该对象的锁，也就意味着要一直等待前一个获得锁的线程离开同步代码块之后才有机会执行。实际上，HashMap是通过哈希函数来确定存放键值对的桶（桶是为了解决哈希冲突而引入的），修改HashMap时并不需要将整个容器锁住，只需要锁住即将修改的“桶”就可以了。HashMap的数据结构如下图所示。  
![](http://i.imgur.com/6CIYRkn.jpg)  
此外，ConcurrentHashMap还提供了原子操作的方法，如下所示：  
- putIfAbsent：如果还没有对应的键值对映射，就将其添加到HashMap中。  
- remove：如果键存在而且值与当前状态相等（equals比较结果为true），则用原子方式移除该键值对映射  
- replace：替换掉映射中元素的原子操作  

**CopyOnWriteArrayList**  
CopyOnWriteArrayList是ArrayList在并发环境下的替代品。CopyOnWriteArrayList通过增加写时复制语义来避免并发访问引起的问题，也就是说任何修改操作都会在底层创建一个列表的副本，也就意味着之前已有的迭代器不会碰到意料之外的修改。这种方式对于不要严格读写同步的场景非常有用，因为它提供了更好的性能。记住，要尽量减少锁的使用，因为那势必带来性能的下降（对数据库中数据的并发访问不也是如此吗？如果可以的话就应该放弃悲观锁而使用乐观锁），CopyOnWriteArrayList很明显也是通过牺牲空间获得了时间（在计算机的世界里，时间和空间通常是不可调和的矛盾，可以牺牲空间来提升效率获得时间，当然也可以通过牺牲时间来减少对空间的使用）。  
![](http://i.imgur.com/91k1sNh.jpg)  
可以通过下面两段代码的运行状况来验证一下CopyOnWriteArrayList是不是线程安全的容器。  
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
class AddThread implements Runnable {
    private List<Double> list;
 
    public AddThread(List<Double> list) {
        this.list = list;
    }
 
    @Override
    public void run() {
        for(int i = 0; i < 10000; ++i) {
            list.add(Math.random());
        }
    }
}
 
public class Test05 {
    private static final int THREAD_POOL_SIZE = 2;
 
    public static void main(String[] args) {
        List<Double> list = new ArrayList<>();
        ExecutorService es = Executors.newFixedThreadPool(THREAD_POOL_SIZE);
        es.execute(new AddThread(list));
        es.execute(new AddThread(list));
        es.shutdown();
    }
}

```  
上面的代码会在运行时产生ArrayIndexOutOfBoundsException，试一试将上面代码25行的ArrayList换成CopyOnWriteArrayList再重新运行。  
```java
List<Double> list = new CopyOnWriteArrayList<>();
```  
**Queue**  
队列是一个无处不在的美妙概念，它提供了一种简单又可靠的方式将资源分发给处理单元（也可以说是将工作单元分配给待处理的资源，这取决于你看待问题的方式）。实现中的并发编程模型很多都依赖队列来实现，因为它可以在线程之间传递工作单元。  
Java 5中的BlockingQueue就是一个在并发环境下非常好用的工具，在调用put方法向队列中插入元素时，如果队列已满，它会让插入元素的线程等待队列腾出空间；在调用take方法从队列中取元素时，如果队列为空，取出元素的线程就会阻塞。  
![](http://i.imgur.com/nKgc8PG.jpg)  
可以用BlockingQueue来实现生产者-消费者并发模型（下一节中有介绍），当然在Java 5以前也可以通过wait和notify来实现线程调度，比较一下两种代码就知道基于已有的并发工具类来重构并发代码到底好在哪里了。  
- 基于wait和notify的实现  

```java
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
/**
 * 公共常量
 * @author 骆昊
 *
 */
class Constants {
    public static final int MAX_BUFFER_SIZE = 10;
    public static final int NUM_OF_PRODUCER = 2;
    public static final int NUM_OF_CONSUMER = 3;
}
 
/**
 * 工作任务
 * @author 骆昊
 *
 */
class Task {
    private String id;  // 任务的编号
 
    public Task() {
        id = UUID.randomUUID().toString();
    }
 
    @Override
    public String toString() {
        return "Task[" + id + "]";
    }
}
 
/**
 * 消费者
 * @author 骆昊
 *
 */
class Consumer implements Runnable {
    private List<Task> buffer;
 
    public Consumer(List<Task> buffer) {
        this.buffer = buffer;
    }
 
    @Override
    public void run() {
        while(true) {
            synchronized(buffer) {
                while(buffer.isEmpty()) {
                    try {
                        buffer.wait();
                    } catch(InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                Task task = buffer.remove(0);
                buffer.notifyAll();
                System.out.println("Consumer[" + Thread.currentThread().getName() + "] got " + task);
            }
        }
    }
}
 
/**
 * 生产者
 * @author 骆昊
 *
 */
class Producer implements Runnable {
    private List<Task> buffer;
 
    public Producer(List<Task> buffer) {
        this.buffer = buffer;
    }
 
    @Override
    public void run() {
        while(true) {
            synchronized (buffer) {
                while(buffer.size() >= Constants.MAX_BUFFER_SIZE) {
                    try {
                        buffer.wait();
                    } catch(InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                Task task = new Task();
                buffer.add(task);
                buffer.notifyAll();
                System.out.println("Producer[" + Thread.currentThread().getName() + "] put " + task);
            }
        }
    }
 
}
 
public class Test06 {
 
    public static void main(String[] args) {
        List<Task> buffer = new ArrayList<>(Constants.MAX_BUFFER_SIZE);
        ExecutorService es = Executors.newFixedThreadPool(Constants.NUM_OF_CONSUMER + Constants.NUM_OF_PRODUCER);
        for(int i = 1; i <= Constants.NUM_OF_PRODUCER; ++i) {
            es.execute(new Producer(buffer));
        }
        for(int i = 1; i <= Constants.NUM_OF_CONSUMER; ++i) {
            es.execute(new Consumer(buffer));
        }
    }
}

```  
- 基于BlockingQueue的实现  

```java
import java.util.UUID;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
 
/**
 * 公共常量
 * @author 骆昊
 *
 */
class Constants {
    public static final int MAX_BUFFER_SIZE = 10;
    public static final int NUM_OF_PRODUCER = 2;
    public static final int NUM_OF_CONSUMER = 3;
}
 
/**
 * 工作任务
 * @author 骆昊
 *
 */
class Task {
    private String id;  // 任务的编号
 
    public Task() {
        id = UUID.randomUUID().toString();
    }
 
    @Override
    public String toString() {
        return "Task[" + id + "]";
    }
}
 
/**
 * 消费者
 * @author 骆昊
 *
 */
class Consumer implements Runnable {
    private BlockingQueue<Task> buffer;
 
    public Consumer(BlockingQueue<Task> buffer) {
        this.buffer = buffer;
    }
 
    @Override
    public void run() {
        while(true) {
            try {
                Task task = buffer.take();
                System.out.println("Consumer[" + Thread.currentThread().getName() + "] got " + task);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
 
/**
 * 生产者
 * @author 骆昊
 *
 */
class Producer implements Runnable {
    private BlockingQueue<Task> buffer;
 
    public Producer(BlockingQueue<Task> buffer) {
        this.buffer = buffer;
    }
 
    @Override
    public void run() {
        while(true) {
            try {
                Task task = new Task();
                buffer.put(task);
                System.out.println("Producer[" + Thread.currentThread().getName() + "] put " + task);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
 
        }
    }
 
}
 
public class Test07 {
 
    public static void main(String[] args) {
        BlockingQueue<Task> buffer = new LinkedBlockingQueue<>(Constants.MAX_BUFFER_SIZE);
        ExecutorService es = Executors.newFixedThreadPool(Constants.NUM_OF_CONSUMER + Constants.NUM_OF_PRODUCER);
        for(int i = 1; i <= Constants.NUM_OF_PRODUCER; ++i) {
            es.execute(new Producer(buffer));
        }
        for(int i = 1; i <= Constants.NUM_OF_CONSUMER; ++i) {
            es.execute(new Consumer(buffer));
        }
    }
}

```  
使用BlockingQueue后代码优雅了很多。  
**并发模型**  
在继续下面的探讨之前，我们还是重温一下几个概念：  
![](http://i.imgur.com/8AMrnTX.jpg)  
重温了这几个概念后，我们可以探讨一下下面的几种并发模型。  
**生产者-消费者**  
一个或多个生产者创建某些工作并将其置于缓冲区或队列中，一个或多个消费者会从队列中获得这些工作并完成之。这里的缓冲区或队列是临界资源。当缓冲区或队列放满的时候，生产这会被阻塞；而缓冲区或队列为空的时候，消费者会被阻塞。生产者和消费者的调度是通过二者相互交换信号完成的。  
**读者-写者**  
当存在一个主要为读者提供信息的共享资源，它偶尔会被写者更新，但是需要考虑系统的吞吐量，又要防止饥饿和陈旧资源得不到更新的问题。在这种并发模型中，如何平衡读者和写者是最困难的，当然这个问题至今还是一个被热议的问题，恐怕必须根据具体的场景来提供合适的解决方案而没有那种放之四海而皆准的方法（不像我在国内的科研文献中看到的那样）。  
**哲学家进餐**  
1965年，荷兰计算机科学家图灵奖得主Edsger Wybe Dijkstra提出并解决了一个他称之为哲学家进餐的同步问题。这个问题可以简单地描述如下：五个哲学家围坐在一张圆桌周围，每个哲学家面前都有一盘通心粉。由于通心粉很滑，所以需要两把叉子才能夹住。相邻两个盘子之间放有一把叉子如下图所示。哲学家的生活中有两种交替活动时段：即吃饭和思考。当一个哲学家觉得饿了时，他就试图分两次去取其左边和右边的叉子，每次拿一把，但不分次序。如果成功地得到了两把叉子，就开始吃饭，吃完后放下叉子继续思考。  
把上面问题中的哲学家换成线程，把叉子换成竞争的临界资源，上面的问题就是线程竞争资源的问题。如果没有经过精心的设计，系统就会出现死锁、活锁、吞吐量下降等问题。  
![](http://i.imgur.com/W568Z6E.jpg)  
下面是用信号量原语来解决哲学家进餐问题的代码，使用了Java 5并发工具包中的Semaphore类（代码不够漂亮但是已经足以说明问题了）。  
```java
//import java.util.concurrent.ExecutorService;
//import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
 
/**
 * 存放线程共享信号量的上下问
 * @author 骆昊
 *
 */
class AppContext {
    public static final int NUM_OF_FORKS = 5;   // 叉子数量(资源)
    public static final int NUM_OF_PHILO = 5;   // 哲学家数量(线程)
 
    public static Semaphore[] forks;    // 叉子的信号量
    public static Semaphore counter;    // 哲学家的信号量
 
    static {
        forks = new Semaphore[NUM_OF_FORKS];
 
        for (int i = 0, len = forks.length; i < len; ++i) {
            forks[i] = new Semaphore(1);    // 每个叉子的信号量为1
        }
 
        counter = new Semaphore(NUM_OF_PHILO - 1);  // 如果有N个哲学家，最多只允许N-1人同时取叉子
    }
 
    /**
     * 取得叉子
     * @param index 第几个哲学家
     * @param leftFirst 是否先取得左边的叉子
     * @throws InterruptedException
     */
    public static void putOnFork(int index, boolean leftFirst) throws InterruptedException {
        if(leftFirst) {
            forks[index].acquire();
            forks[(index + 1) % NUM_OF_PHILO].acquire();
        }
        else {
            forks[(index + 1) % NUM_OF_PHILO].acquire();
            forks[index].acquire();
        }
    }
 
    /**
     * 放回叉子
     * @param index 第几个哲学家
     * @param leftFirst 是否先放回左边的叉子
     * @throws InterruptedException
     */
    public static void putDownFork(int index, boolean leftFirst) throws InterruptedException {
        if(leftFirst) {
            forks[index].release();
            forks[(index + 1) % NUM_OF_PHILO].release();
        }
        else {
            forks[(index + 1) % NUM_OF_PHILO].release();
            forks[index].release();
        }
    }
}
 
/**
 * 哲学家
 * @author 骆昊
 *
 */
class Philosopher implements Runnable {
    private int index;      // 编号
    private String name;    // 名字
 
    public Philosopher(int index, String name) {
        this.index = index;
        this.name = name;
    }
 
    @Override
    public void run() {
        while(true) {
            try {
                AppContext.counter.acquire();
                boolean leftFirst = index % 2 == 0;
                AppContext.putOnFork(index, leftFirst);
                System.out.println(name + "正在吃意大利面（通心粉）...");   // 取到两个叉子就可以进食
                AppContext.putDownFork(index, leftFirst);
                AppContext.counter.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
 
public class Test04 {
 
    public static void main(String[] args) {
        String[] names = { "骆昊", "王大锤", "张三丰", "杨过", "李莫愁" };   // 5位哲学家的名字
//      ExecutorService es = Executors.newFixedThreadPool(AppContext.NUM_OF_PHILO); // 创建固定大小的线程池
//      for(int i = 0, len = names.length; i < len; ++i) {
//          es.execute(new Philosopher(i, names[i]));   // 启动线程
//      }
//      es.shutdown();
        for(int i = 0, len = names.length; i < len; ++i) {
            new Thread(new Philosopher(i, names[i])).start();
        }
    }
 
}

```  
现实中的并发问题基本上都是这三种模型或者是这三种模型的变体。  
**测试并发代码**  
对并发代码的测试也是非常棘手的事情，棘手到无需说明大家也很清楚的程度，所以这里我们只是探讨一下如何解决这个棘手的问题。我们建议大家编写一些能够发现问题的测试并经常性的在不同的配置和不同的负载下运行这些测试。不要忽略掉任何一次失败的测试，线程代码中的缺陷可能在上万次测试中仅仅出现一次。具体来说有这么几个注意事项：  
- 不要将系统的失效归结于偶发事件，就像拉不出屎的时候不能怪地球没有引力。  
- 先让非并发代码工作起来，不要试图同时找到并发和非并发代码中的缺陷。  
- 编写可以在不同配置环境下运行的线程代码。  
- 编写容易调整的线程代码，这样可以调整线程使性能达到最优。  
- 让线程的数量多于CPU或CPU核心的数量，这样CPU调度切换过程中潜在的问题才会暴露出来。  
- 让并发代码在不同的平台上运行。  
- 通过自动化或者硬编码的方式向并发代码中加入一些辅助测试的代码。  

**Java 7的并发编程**  
Java 7中引入了TransferQueue，它比BlockingQueue多了一个叫transfer的方法，如果接收线程处于等待状态，该操作可以马上将任务交给它，否则就会阻塞直至取走该任务的线程出现。可以用TransferQueue代替BlockingQueue，因为它可以获得更好的性能。  
刚才忘记了一件事情，Java 5中还引入了Callable接口、Future接口和FutureTask接口，通过他们也可以构建并发应用程序，代码如下所示。  
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
 
public class Test07 {
    private static final int POOL_SIZE = 10;
 
    static class CalcThread implements Callable<Double> {
        private List<Double> dataList = new ArrayList<>();
 
        public CalcThread() {
            for(int i = 0; i < 10000; ++i) {
                dataList.add(Math.random());
            }
        }
 
        @Override
        public Double call() throws Exception {
            double total = 0;
            for(Double d : dataList) {
                total += d;
            }
            return total / dataList.size();
        }
 
    }
 
    public static void main(String[] args) {
        List<Future<Double>> fList = new ArrayList<>();
        ExecutorService es = Executors.newFixedThreadPool(POOL_SIZE);
        for(int i = 0; i < POOL_SIZE; ++i) {
            fList.add(es.submit(new CalcThread()));
        }
 
        for(Future<Double> f : fList) {
            try {
                System.out.println(f.get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
 
        es.shutdown();
    }
}

```  
Callable接口也是一个单方法接口，显然这是一个回调方法，类似于函数式编程中的回调函数，在Java 8 以前，Java中还不能使用Lambda表达式来简化这种函数式编程。和Runnable接口不同的是Callable接口的回调方法call方法会返回一个对象，这个对象可以用将来时的方式在线程执行结束的时候获得信息。上面代码中的call方法就是将计算出的10000个0到1之间的随机小数的平均值返回，我们通过一个Future接口的对象得到了这个返回值。目前最新的Java版本中，Callable接口和Runnable接口都被打上了@FunctionalInterface的注解，也就是说它可以用函数式编程的方式（Lambda表达式）创建接口对象。  
下面是Future接口的主要方法：  
- get()：获取结果。如果结果还没有准备好，get方法会阻塞直到取得结果；当然也可以通过参数设置阻塞超时时间。  
- cancel()：在运算结束前取消。  
- isDone()：可以用来判断运算是否结束。  

Java 7中还提供了分支/合并（fork/join）框架，它可以实现线程池中任务的自动调度，并且这种调度对用户来说是透明的。为了达到这种效果，必须按照用户指定的方式对任务进行分解，然后再将分解出的小型任务的执行结果合并成原来任务的执行结果。这显然是运用了分治法（divide-and-conquer）的思想。下面的代码使用了分支/合并框架来计算1到10000的和，当然对于如此简单的任务根本不需要分支/合并框架，因为分支和合并本身也会带来一定的开销，但是这里我们只是探索一下在代码中如何使用分支/合并框架，让我们的代码能够充分利用现代多核CPU的强大运算能力。  
```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;
 
class Calculator extends RecursiveTask<Integer> {
    private static final long serialVersionUID = 7333472779649130114L;
 
    private static final int THRESHOLD = 10;
    private int start;
    private int end;
 
    public Calculator(int start, int end) {
        this.start = start;
        this.end = end;
    }
 
    @Override
    public Integer compute() {
        int sum = 0;
        if ((end - start) < THRESHOLD) {    // 当问题分解到可求解程度时直接计算结果
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            int middle = (start + end) >>> 1;
            // 将任务一分为二
            Calculator left = new Calculator(start, middle);
            Calculator right = new Calculator(middle + 1, end);
            left.fork();
            right.fork();
            // 注意：由于此处是递归式的任务分解，也就意味着接下来会二分为四，四分为八...
 
            sum = left.join() + right.join();   // 合并两个子任务的结果
        }
        return sum;
    }
 
}
 
public class Test08 {
 
    public static void main(String[] args) throws Exception {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        Future<Integer> result = forkJoinPool.submit(new Calculator(1, 10000));
        System.out.println(result.get());
    }
}
```  
伴随着Java 7的到来，Java中默认的数组排序算法已经不再是经典的快速排序（双枢轴快速排序）了，新的排序算法叫TimSort，它是归并排序和插入排序的混合体，TimSort可以通过分支合并框架充分利用现代处理器的多核特性，从而获得更好的性能（更短的排序时间）。  

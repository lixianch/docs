## 并发 ##
### 线程基础 ###
**定义任务** 
线程可以驱动任务，因此需要一种描述任务的方式，可以通过实现Runnable接口来提供。要想定义任务只需实现Runnable接口，编写run方法，这个方法并无特殊之处——它不会产生任何内在的线程能力。要想实现线程行为，必须显示地将其附着到线程之上。
```java
public class LiftOff implements Runnable {
    private static Integer taskId=0;
    protected Integer countDown = 10;
    private final Integer id = taskId++;
    public LiftOff(){}
    public LiftOff(Integer countDown){
        this.countDown = countDown;
    }
    public String status(){
        return "#"+id+"("+(countDown > 0? countDown : "LiftOff")+")";
    }
    @Override
    public void run() {
        while(countDown-- > 0){
            System.out.println(status());
            Thread.yield();
        }
    }
}
```
**Thread类**   
将Runnable对象转变为工作任务的传统方式是把它提交给一个Thread构造器，然后调用Thread的run方法，启动线程。
```java
public class BasicThreads {
    public static void main(String[] args){
        Thread t = new Thread(new LiftOff());
        t.start();
        System.out.println("waiting for LiftOff");
    }
}
```
**使用Executor**  
Java SE5的java.util.concurrent包中的执行器(Executor)可以管理Thread对象。Executor可以管理异步任务的执行，而无须显示地管理线程的生命周期。Executor在Java SE5后是启动任务的优选方法。  
***执行器使用完毕后一定要调用shutdown()方法手动关闭。***  
创建执行器方法：
```java
ExecutorService exec = Executors.newCachedThreadPool();
```  
执行器种类：  

执行器名称|描述 
---|---
CachedThreadPool|CachedThreadPool在程序执行过程中通常会创建与所需数量相同的线程，然后在它回收旧线程时停止创建新线程，因此它是合理的Executor的首选。   
FixedThreadPool|FixedThreadPool可以一次性预先执行代价高昂的线程分配，因此也就可以限制线程的数量。这可以节省时间，因为不用为每个任务都固定地付出创建线程的开销。在事件驱动的系统中，需要线程的事件处理器，通过直接从池中获取线程，也可以尽快得到服务。不会滥用可获得的资源，因为FixedThreadPool使用的Thread对象的数量是有界的。  
SingleThreadExecutor|SingleThreadExecutor是线程数量为1的FixedThreadPool。SingleThreadExector会序列化所有提交给它的任务，并会维护它自己(隐藏)的悬挂任务队列。  
  
**从任务中产生返回值**  
从任务中产生返回值可以通过实现Callable接口实现，在Java SE5中引入的Callable是一种具有类型参数的泛型，它的类型参数表示的是从call()中返回的值，并且必须使用ExecutorService.submit()方法调用它。
```java
public class CallableDemo {


    public static void main(String[] args){
        ExecutorService exec = Executors.newCachedThreadPool();
        List<Future<String>> results = new ArrayList<Future<String>>();
        for(int i = 0; i < 5; i++){
            results.add(exec.submit(new TaskWithResult(i)));
        }

        try {
            for(Future<String> r : results){
                System.out.println(r.get());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        } finally {
            exec.shutdown();
        }
    }
}

class TaskWithResult implements Callable<String>{
    private Integer id;
    public TaskWithResult(Integer id){
        this.id = id;
    }
    @Override
    public String call() throws Exception {
        return "TaskWithResult-"+id+" result";
    }
}
```   

**休眠**  
两种方式：
1. 调用线程sleep()方法
2. 使用TimeUnit `TimeUnit.MICROSECONDS.sleep(100);`
**注意：休眠时需要捕获InterruptedException异常** 
 
**优先级**
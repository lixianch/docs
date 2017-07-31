Servlet 3.0之前，这些长期运行的线程容器特定的解决方案，我们可以产生一个单独的工作线程完成耗时的任务，然后返回响应客户。Servlet线程返回Servlet池后启动工作线程。Tomcat 的 Comet、WebLogic FutureResponseServlet 和 WebSphere Asynchronous Request Dispatcher都是实现异步处理的很好示例。

容器特定解决方案的问题在于，在不改变应用程序代码时不能移动到其他Servlet容器。这就是为什么在Servlet3.0提供标准的方式异步处理Servlet的同时增加异步Servlet支持。  
进一步理解：  
AsyncContext不是让你异步输出，而是让你同步输出，但是解放服务器端的线程使用，使用AsyncContext的时候，对于浏览器来说，他们是同步在等待输出的，但是对于服务器端来说，处理此请求的线程并没有卡在那里等待，则是把当前的处理转为线程池处理了，关键就在于线程池，服务器端会起一个线程池去服务那些需要异步处理的请求，而如果你自己每次请求去起一个线程处理的话，这就有可能会耗大量的线程。  
### 实现异步Servlet ###

1. 首先Servlet,我们提供异步支持 Annotation @WebServlet  的属性asyncSupported 值为true。   
2. 由于实际实现委托给另一个线程，我们应该有一个线程池实现。我们可以一个通过Executors framework 创建线程池和使用servlet context listener来初始化线程池。   
3. 通过ServletRequest.startAsync方法获取AsyncContext的实例。AsyncContext提供方法让ServletRequest和ServletResponse对象引用。它还提供了使用调度方法将请求转发到另一个 dispatch() 方法。  
4. 编写一个可运行的实现,我们将进行重处理，然后使用AsyncContext对象发送请求到另一个资源或使用ServletResponse编写响应对象。一旦处理完成，我们通过AsyncContext.complete()方法通知容器异步处理完成。  
5. 添加AsyncListener实现AsyncContext对象实现回调方法，我们可以使用它来提供错误响应客户端装进箱的错误或超时，而异步线程处理。在这里我们也可以做一些清理工作。  

##### 在监听中初始化线程池 #####  
 ```java   
package com.journaldev.servlet.async;  
   
import java.util.concurrent.ArrayBlockingQueue;  
import java.util.concurrent.ThreadPoolExecutor;  
import java.util.concurrent.TimeUnit;  
   
import javax.servlet.ServletContextEvent;  
import javax.servlet.ServletContextListener;  
import javax.servlet.annotation.WebListener;  
   
@WebListener  
public class AppContextListener implements ServletContextListener {  
   
public void contextInitialized(ServletContextEvent servletContextEvent) {  
   
// create the thread pool  
ThreadPoolExecutor executor = new ThreadPoolExecutor(100, 200, 50000L,  
TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(100));  
servletContextEvent.getServletContext().setAttribute("executor",  
executor);  
   
}  
   
public void contextDestroyed(ServletContextEvent servletContextEvent) {  
ThreadPoolExecutor executor = (ThreadPoolExecutor) servletContextEvent  
.getServletContext().getAttribute("executor");  
executor.shutdown();  
}  
   
}  
``` 
##### 工作线程实现 #####  
```java
package com.journaldev.servlet.async;  
   
import java.io.IOException;  
import java.io.PrintWriter;  
   
import javax.servlet.AsyncContext;  
   
public class AsyncRequestProcessor implements Runnable {  
   
    private AsyncContext asyncContext;  
    private int secs;  
   
    public AsyncRequestProcessor() {  
    }  
   
    public AsyncRequestProcessor(AsyncContext asyncCtx, int secs) {  
        this.asyncContext = asyncCtx;  
        this.secs = secs;  
    }  
   
    @Override  
    public void run() {  
        System.out.println("Async Supported? "  
                + asyncContext.getRequest().isAsyncSupported());  
        longProcessing(secs);  
        try {  
            PrintWriter out = asyncContext.getResponse().getWriter();  
            out.write("Processing done for " + secs + " milliseconds!!");  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
        //complete the processing  
        asyncContext.complete();  
    }  
   
    private void longProcessing(int secs) {  
        // wait for given time before finishing  
        try {  
            Thread.sleep(secs);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}  
``` 
**注意:** 在请求和响应时使用AsyncContext对象，然后在完成时调用 asyncContext.complete() 方法。    
##### AsyncListener 实现 #####  
```java
package com.journaldev.servlet.async;  
   
import java.io.IOException;  
import java.io.PrintWriter;  
   
import javax.servlet.AsyncEvent;  
import javax.servlet.AsyncListener;  
import javax.servlet.ServletResponse;  
import javax.servlet.annotation.WebListener;  
   
@WebListener  
public class AppAsyncListener implements AsyncListener {  
   
    @Override  
    public void onComplete(AsyncEvent asyncEvent) throws IOException {  
        System.out.println("AppAsyncListener onComplete");  
        // we can do resource cleanup activity here  
    }  
   
    @Override  
    public void onError(AsyncEvent asyncEvent) throws IOException {  
        System.out.println("AppAsyncListener onError");  
        //we can return error response to client  
    }  
   
    @Override  
    public void onStartAsync(AsyncEvent asyncEvent) throws IOException {  
        System.out.println("AppAsyncListener onStartAsync");  
        //we can log the event here  
    }  
   
    @Override  
    public void onTimeout(AsyncEvent asyncEvent) throws IOException {  
        System.out.println("AppAsyncListener onTimeout");  
        //we can send appropriate response to client  
        ServletResponse response = asyncEvent.getAsyncContext().getResponse();  
        PrintWriter out = response.getWriter();  
        out.write("TimeOut Error in Processing");  
    }  
   
}
```  
通知的实现在 Timeout()方法，通过它发送超时响应给客户端。  
##### Async Servlet 实现 #####  
现在，当我们将上面运行servlet URL：

http://localhost:8080/AsyncServletExample/AsyncLongRunningServlet?time=8000

得到响应和日志：

```
AsyncLongRunningServlet Start::Name=http-bio-8080-exec-50::ID=124  
AsyncLongRunningServlet End::Name=http-bio-8080-exec-50::ID=124::Time Taken=1 ms.  
Async Supported? true  
AppAsyncListener onComplete  
```  
如果运行时设置time=9999，在客户端超时以后会得到响应超时错误处理和日志：  
```
AsyncLongRunningServlet Start::Name=http-bio-8080-exec-44::ID=117  
AsyncLongRunningServlet End::Name=http-bio-8080-exec-44::ID=117::Time Taken=1 ms.  
Async Supported? true  
AppAsyncListener onTimeout  
AppAsyncListener onError  
AppAsyncListener onComplete  
Exception in thread "pool-5-thread-6" java.lang.IllegalStateException: The request associated with the AsyncContext has already completed processing.  
    at org.apache.catalina.core.AsyncContextImpl.check(AsyncContextImpl.java:439)  
    at org.apache.catalina.core.AsyncContextImpl.getResponse(AsyncContextImpl.java:197)  
    at com.journaldev.servlet.async.AsyncRequestProcessor.run(AsyncRequestProcessor.java:27)  
    at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)  
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)  
    at java.lang.Thread.run(Thread.java:680) 
```  
**注意：** Servlet线程执行完，很快就和所有主要的处理工作是发生在其他线程。

##### Servlet 3.0 新特性概述 #####  
异步处理支持：有了该特性，Servlet 线程不再需要一直阻塞，直到业务处理完毕才能再输出响应，最后才结束该 Servlet 线程。在接收到请求之后，Servlet 线程可以将耗时的操作委派给另一个线程来完成，自己在不生成响应的情况下返回至容器。针对业务处理较耗时的情况，这将大大减少服务器资源的占用，并且提高并发处理速度。  
##### Servlet 3.0异步处理支持 #####  
Servlet 3.0 之前，一个普通 Servlet 的主要工作流程大致如下：首先，Servlet 接收到请求之后，可能需要对请求携带的数据进行一些预处理；接着，调用业务接口的某些方法，以完成业务处理；最后，根据处理的结果提交响应，Servlet 线程结束。其中第二步的业务处理通常是最耗时的，这主要体现在数据库操作，以及其它的跨网络调用等，在此过程中，Servlet 线程一直处于阻塞状态，直到业务方法执行完毕。在处理业务的过程中，Servlet 资源一直被占用而得不到释放，对于并发较大的应用，这有可能造成性能的瓶颈。对此，在以前通常是采用私有解决方案来提前结束 Servlet 线程，并及时释放资源。  
Servlet 3.0 针对这个问题做了开创性的工作，现在通过使用 Servlet 3.0 的异步处理支持，之前的 Servlet 处理流程可以调整为如下的过程：首先，Servlet 接收到请求之后，可能首先需要对请求携带的数据进行一些预处理；接着，Servlet 线程将请求转交给一个异步线程来执行业务处理，线程本身返回至容器，此时 Servlet 还没有生成响应数据，异步线程处理完业务以后，可以直接生成响应数据（异步线程拥有 ServletRequest 和 ServletResponse 对象的引用），或者将请求继续转发给其它 Servlet。如此一来， Servlet 线程不再是一直处于阻塞状态以等待业务逻辑的处理，而是启动异步线程之后可以立即返回。  
##### Servlet 3.0异步处理调用 #####  
Servlet 3.0的异步处理是通过AsyncContext类来处理的，Servlet可通过ServletRequest的如下两个方法开启异步调用、创建AsyncContext对象：   
AsyncContext startAsync()   
AsyncContext startAsync(ServletRequest, ServletResponse)   
重复调用上面的方法将得到同一个AsyncContext对象。AsyncContext对象代表异步处理的上下文，它提供了一些工具方法，可完成设置异步调用的超时时长，dispatch用于请求、启动后台线程、获取request、response对象等功能。  
除此之外，Servlet 3.0 还为异步处理提供了一个监听器，使用 AsyncListener 接口表示。它可以监控如下四种事件：  
异步线程开始时，调用 AsyncListener 的 onStartAsync(AsyncEvent event) 方法；  
异步线程出错时，调用 AsyncListener 的 onError(AsyncEvent event) 方法；  
异步线程执行超时，调用 AsyncListener 的 onTimeout(AsyncEvent event) 方法；  
异步执行完毕时，调用 AsyncListener 的 onComplete(AsyncEvent event) 方法；  

	  

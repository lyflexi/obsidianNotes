默认情况下，Java 进程需要等待所有线程都运行结束，才会结束，否则进程不会终止。

有一种特殊的线程叫做守护线程，只要其它主线程运行结束了，即使守护线程的代码没有执行完也会强制结束，整个进程终止。
```java
/**  
 * @Author: ly  
 * @Date: 2024/3/14 17:13  
 */@Slf4j(topic = "c.Test15")  
public class DaemonTest {  
    public static void main(String[] args) throws InterruptedException {  
        Thread t1 = new Thread(() -> {  
            while (true) {  
                if (Thread.currentThread().isInterrupted()) {  
                    break;  
                }  
                log.debug("子线程正在循环");  
            }  
            log.debug("结束");  
        }, "t1");  
        t1.setDaemon(true);  
        t1.start();  
  
        Thread.sleep(1000);  
        log.debug("结束");  
    }  
}
```
打印信息：
```shell
17:16:55.800 c.Test15 [main] - 结束
17:16:55.798 c.Test15 [t1] - 子线程正在循环
17:16:55.800 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环
17:16:55.801 c.Test15 [t1] - 子线程正在循环

Process finished with exit code 0
```
守护线程其实也经常能够见到：
- 垃圾回收器线程就是一种守护线程 ，因此主程序提前结束会强行终止gc线程
- 另外，Tomcat 中的 Acceptor 和 Poller 线程都是守护线程，所以 Tomcat 接收到 shutdown 命令后，不会等待它们处理完当前请求
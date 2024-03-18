固定运行顺序 比如，必须先 2 后 1 打印
```java
package org.lyflexi.multiThread.designParten.syncParten.fixedSequence.byLockSupport;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.LockSupport;  
  
@Slf4j(topic = "c.FixedSequenceByLockSupport")  
public class FixedSequenceByLockSupport {  
    public static void main(String[] args) {  
  
        Thread t1 = new Thread(() -> {  
            LockSupport.park();  
            log.debug("1");  
        }, "t1");  
        t1.start();  
  
        new Thread(() -> {  
            log.debug("2");  
            LockSupport.unpark(t1);  
        },"t2").start();  
    }  
}
```
打印信息如下：
```shell
20:57:36.936 c.FixedSequenceByLockSupport [t2] - 2
20:57:36.939 c.FixedSequenceByLockSupport [t1] - 1

Process finished with exit code 0
```
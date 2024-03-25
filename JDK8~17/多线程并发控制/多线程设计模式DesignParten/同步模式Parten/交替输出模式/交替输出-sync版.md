线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。要求输出 abcabcabcabcabc 怎么实现

面向对象封装WaitNotify类
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.bySync;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 21:23  
 */  
/*  
输出内容       等待标记     下一个标记  
   a           1             2   
   b           2             3  
   c           3             1 */
class WaitNotify {  

    // 等待标记  
    private int flag; // 2  
    // 循环次数  
    private int loopNumber;  
  
    public WaitNotify(int flag, int loopNumber) {  
        this.flag = flag;  
        this.loopNumber = loopNumber;  
    }  

    // 打印               a           1             2    
    public void print(String str, int waitFlag, int nextFlag) {  
        for (int i = 0; i < loopNumber; i++) {  
            synchronized (this) {  
                while(flag != waitFlag) {  
                    try {  
                        this.wait();  //可以直接使用this.wait真是巧妙呀
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.print(str);  
                flag = nextFlag;  
                this.notifyAll();  
            }  
        }  
    }  
  

}
```
测试类TestByWaitNotify
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.bySync;  
  
import lombok.extern.slf4j.Slf4j;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 21:23  
 */
@Slf4j(topic = "c.TestByWaitNotify")  
public class TestByWaitNotify {  
    public static void main(String[] args) {  
        WaitNotify wn = new WaitNotify(1, 5); 
        new Thread(() -> {  
            wn.print("a", 1, 2);//因为上面new WaitNotify(1, 5)，所以"a", 1, 2先满足条件启动
        }).start();  
        new Thread(() -> {  
            wn.print("b", 2, 3);  
        }).start();  
        new Thread(() -> {  
            wn.print("c", 3, 1);  
        }).start();  
    }  
}
```
打印信息如下：
```shell
abcabcabcabcabc
Process finished with exit code 0

```
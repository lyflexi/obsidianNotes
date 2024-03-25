在没有利用 cpu 来计算时，不要让 while(true) 空转浪费 cpu，这时可以使用 yield 或 sleep 来让出 cpu 的使用权 给其他程序 
```java
while(true) {  
    try {  
        Thread.sleep(50);  
	} catch (InterruptedException e) {  
        e.printStackTrace();  
	}  
}
```

还可以用 wait 或 条件变量达到类似的效果 ，不同的是wait 或 条件变量都需要在锁内使用，并且需要相应的唤醒操作，一般适用于要进行同步的场景 

而sleep 适用于无需锁的场景
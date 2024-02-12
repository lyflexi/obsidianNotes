# Serial回收器

Serial收集器是最基本、历史最悠久的垃圾收集器了。
Serial收集器作为HotSpot中Client模式下的默认新生代垃圾收集器。在JDK1.3之前是回收新生代的唯一选择。

- Serial收集器采用标记复制算法
    
- Serial收集器采用串行回收
    
- Serial收集器采用"Stop一 the一World"机制。Serial收集器是一个单线程的收集器，在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束（Stop The World ）。
    

# Serial Old收集器

Serial Old在Client模式下，是默认的老年代的垃圾回收器。
Serial Old收集器同样也采用了串行回收 和"Stop the World"机制。只不过内存回收算法使用的是标记压缩（标记整理）算法。

![[Pasted image 20231226120231.png]]
LinkedList类的结构
```java
public class LinkedList<E> extends AbstractSequentialList<E> 
	implements List<E>, Deque<E>, Cloneable, Serializable
```

LinkedList是的双向链表。实现了Deque(双端队列)接口，也就实现了Deque接口中的抽象方法，所以它可以被当作栈、队列进行操作。
# LinkedList 当做队列的使用
LinkedList作为先进先出的队列使用时，出队列使用poll()方法，入队列建议使用offer()而不建议使用add()，原因如下：
- add(E e) 方法：  如果队列的容量已满，在尝试将元素添加到队列时，add 方法会抛出 IllegalStateException 异常。  
- offer(E e) 方法：  如果队列的容量已满，在尝试将元素添加到队列时，offer 方法会返回 false，避免了在队列满时抛出异常的情况。 
```java
//定义
LinkedList<Integer> queue = new LinkedList<Integer>();

//入队
queue.offer(1);
//出队
queue.poll();
```
# LinkedList 当做栈列使用
作为先进后出的栈时，使用push()方法向栈中添加元素，使用pop()方法取出栈顶元素，使用peek()方法检查栈顶元素
```java
//定义栈
LinkedList<Integer> stack = new LinkedList<Integer>();

//push元素
stack.push(1)
//pop元素
stack.pop()
//获取栈顶元素，不弹出
stack.peek()

```

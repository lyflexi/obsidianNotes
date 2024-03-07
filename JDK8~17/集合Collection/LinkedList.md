LinkedList类的结构
```java
public class LinkedList<E> extends AbstractSequentialList<E> 
	implements List<E>, Deque<E>, Cloneable, Serializable
```

LinkedList是的双向链表。实现了Deque(双端队列)接口，也就实现了Deque接口中的抽象方法，所以它可以被当作栈、队列进行操作。

# LinkedList 当做队列的使用
LinkedList作为先进先出的队列使用时，使用offer() 方法在队尾插入元素，poll()取出队首元素。
```java
//定义
LinkedList<Integer> queue = new LinkedList<Integer>();

//添加元素
queue.add(1);

//删除队列头元素
queue.poll();

//获取队列头元素，不删除
queue.peek();

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

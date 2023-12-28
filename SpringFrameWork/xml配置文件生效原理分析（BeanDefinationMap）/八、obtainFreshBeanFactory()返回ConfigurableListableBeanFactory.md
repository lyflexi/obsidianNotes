接下来该执行getBeanFactory()了
![[Pasted image 20231228202501.png]]

getBeanFactory()位于AbstractRefreshableApplicationContext类
```java
/** Bean factory for this context. */  
@Nullable  
private volatile DefaultListableBeanFactory beanFactory;


@Override  
public final ConfigurableListableBeanFactory getBeanFactory() {  
    DefaultListableBeanFactory beanFactory = this.beanFactory;  
    if (beanFactory == null) {  
       throw new IllegalStateException("BeanFactory not initialized or already closed - " +  
             "call 'refresh' before accessing beans via the ApplicationContext");  
    }  
    return beanFactory;  
}
```

最终返回的是ConfigurableListableBeanFactory类型的beanFactory
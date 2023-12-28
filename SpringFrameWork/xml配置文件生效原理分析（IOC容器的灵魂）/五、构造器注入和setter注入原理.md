# 构造器注入原理parseConstructorArgElements
1、我对第一节中的A类做一个小小的修改，在构造方法中注入一个name属性：
```java
public class A {
    private String name;

    /**
     * 构造注入
     *
     * @param name
     */
    A(String name) {
        this.name = name;
    }


    @Override
    public String toString() {
        return "I am " + name;
    }
}

```

2、在xml中做对应的修改：
```xml
    <!--    定义一个A的bean，默认是单例，并通过构造器注入name属性值-->
    <bean id="a" class="com.sise.course1.A">
        <constructor-arg name="name" value="A class"></constructor-arg>
    </bean>
```

3、看到执行结果，spring通过构造注入已经把name的值给注入进去了。
![[Pasted image 20231228165016.png]]
4、接下来AbstractBeanDefinition bd = createBeanDefinition(className, parent)执行结束
![[Pasted image 20231228165204.png]]
来到了parseConstructorArgElements方法（BeanDefinitionParserDelegate类的方法，用于解析bean标签下的constructor-arg标签）
```java
	public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			//循环遍历所有的constructor-arg标签
			if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) {
				parseConstructorArgElement((Element) node, bd);
			}
		}
	}

```
5、parseConstructorArgElement方法就是把xml的构造参数注入到BeanDefinition对象，我们说的构造注入其实最先注入的是BeanDefinition
![[Pasted image 20231228165550.png]]
6、结语：本节给大家展示了spring中xml是如何通过构造器注入的以及构造器的注入本质就是往BeanDefinition注入，因为这步对spring的用户来说是不可见的，所以就以为注入是直接注入到bean里面。
# setter注入原理parsePropertyElements
1、xml的setter注入和构造器的注入原理相似，先对xml和Aclass修改成setter注入
```java
public class A {
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "I am " + name;
    }
}
```

```xml

    <!--    定义一个A的bean，默认是单例，并通过setter注入name属性值-->
    <bean id="a" class="com.sise.course1.A" >
        <property name = "name" value="A class"></property>
    </bean>
```

2、运行结果如下，可以看到成功注入
![[Pasted image 20231228165759.png]]
3、回到我们解析标签的parseBeanDefinitionElement方法，他调用了parsePropertyElements方法进行setter参数注入，其原理与parseConstructorArgElements类似
```java
	public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
			   //循环遍历解析property标签
				parsePropertyElement((Element) node, bd);
			}
		}
	}
```

4、与构造注入不同的是，setter属性注入时创建的是PropertyValue对象，但最终这些注入的属性对象都添加到BeanDefinition上
![[Pasted image 20231228170025.png]]
5、结语：由于xml的配置方式已经被弃用，所以本节不对property标签的属性（如：ref、description等等）作详细讲解，了解个大概的注入原理即可。

提到FactoryBean，就不得不与BeanFactory比较一番。
- BeanFactory : 是 Factory， IOC容器或者对象工厂，所有的Bean都由它进行管理
- FactoryBean : 是Bean ，是一个能产生或者修饰对象生成的工厂 Bean，实现与工厂模式和修饰器模式类似

一般情况下，Spring是通过反射机制利用Bean的class属性指定实现类来实例化Bean的。在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，那么则需要在标签中提供大量的配置信息，配置方式的灵活性是受限的，这时采用编码的方式可以得到一个更加简单的方案。Spring为此提供了一个org.springframework.bean.factory.FactoryBean的接口，用户可以通过实现该接口通过编码的方式定制Bean的逻辑。FactoryBean接口对于Spring框架来说占有非常重要的地位，==Spring自身就提供了70多个FactoryBean接口的实现==。它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。从Spring 3.0开始，FactoryBean开始支持泛型，即接口声明改为`FactoryBean<T>`的形式。在Spring 4.3.12.RELEASE这个版本中，FactoryBean接口的定义如下所示。

```Java
/*
 * Copyright 2002-2018 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.beans.factory;

import org.springframework.lang.Nullable;

public interface FactoryBean<T> {

        @Nullable
        T getObject() throws Exception;


        @Nullable
        Class<?> getObjectType();


        default boolean isSingleton() {
                return true;
        }

}
```

- T getObject()：返回由FactoryBean创建的bean实例，如果isSingleton()返回true，那么该实例会放到Spring容器中单实例缓存池中
- boolean isSingleton()：返回由FactoryBean创建的bean实例的作用域是singleton还是prototype
- Class getObjectType()：返回FactoryBean创建的Bean实例的类型

那么FactoryBean是如何实现Bean注入的呢？首先定义实现了FactoryBean接口的类

```Java
public class TeacherFactoryBean implements FactoryBean<Teacher> {
     /**
      * 返回此工厂管理的对象实例
      /
     @Override
     public Teacher getObject() throws Exception {
      return new Teacher();
     }
    // 是单例吗？
    // 如果返回true，那么代表这个bean是单实例，在容器中只会保存一份；
    // 如果返回false，那么代表这个bean是多实例，每次获取都会创建一个新的bean
    @Override
    public boolean isSingleton() {
            // TODO Auto-generated method stub
            return false;
    }
     /
      * 返回此 FactoryBean 创建的对象
      **/
     @Override
     public Class<?> getObjectType() {
      return Teacher.class;
     }
}
```

然后通过 @Configuration + @Bean的方式将TeacherFactoryBean加入到容器中。
注意：我们没有向容器中注入Teacher, 而是直接注入的TeacherFactoryBean，此时通过applicationContext.getBean("teacherFactoryBean")方法返回的不是FactoryBean本身，而是FactoryBean#getObject()方法所返回的对象，getObject()方法代理了getBean()方法。

```Java
@Configuration
public class MyConfig {
    @Bean
    public TeacherFactoryBean teacherFactoryBean(){
            return new TeacherFactoryBean();
    }
}
```

如果想要获取FactoryBean本身，只需要在Bean的id前面加上&符号即可，BeanFactory源码如下：
![[Pasted image 20231226150011.png]]
传统的Web请求处理的方式是@Controller + @RequestMapping：耦合式 （路由与@Autowired业务耦合）  


SpringMVC 5.2 以后 允许我们对Restfull风格的请求使用函数式Web的方式来定义Web的请求处理流程，这意味着我们可以集中式管理web-api，将路由与业务代码完全分离
# 核心API

- **RouterFunction**
- **RequestPredicate**
- **ServerRequest**
- **ServerResponse**

场景模拟：User RESTful - CRUD  
- GET /user/1 获取1号用户  
- GET /users 获取所有用户  
- POST /user 请求体携带JSON，新增一个用户  
- PUT /user/1 请求体携带JSON，修改1号用户  
- DELETE /user/1 删除1号用户  
# 集中化路由配置
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.servlet.function.RequestPredicate;
import org.springframework.web.servlet.function.RouterFunction;
import org.springframework.web.servlet.function.ServerResponse;

import static org.springframework.web.servlet.function.RequestPredicates.accept;
import static org.springframework.web.servlet.function.RouterFunctions.route;

@Configuration(proxyBeanMethods = false)
public class MyRoutingConfiguration {

    private static final RequestPredicate ACCEPT_JSON = accept(MediaType.APPLICATION_JSON);
	@Bean  
	public RouterFunction<ServerResponse> userRoute(UserBizHandler userBizHandler/*这个会被自动注入进来*/){  
	  
	    return RouterFunctions.route() //开始定义路由信息  
	            .GET("/user/{id}", RequestPredicates.accept(MediaType.ALL), userBizHandler::getUser)  
	            .GET("/users", userBizHandler::getUsers)  
	            .POST("/user", RequestPredicates.accept(MediaType.APPLICATION_JSON), userBizHandler::saveUser)  
	            .PUT("/user/{id}", RequestPredicates.accept(MediaType.APPLICATION_JSON), userBizHandler::updateUser)  
	            .DELETE("/user/{id}", userBizHandler::deleteUser)  
	            .build();  
	}

}


```

# 业务类
```java
package org.lyflexi.debug_springboot.functionweb;  

  

@Service  
public class UserBizHandler {  
  
    /**  
     * 查询指定id的用户  
     * @param request  
     * @return  
     */  
    public ServerResponse getUser(ServerRequest request) throws Exception{  
        String id = request.pathVariable("id");  
        log.info("查询 【{}】 用户信息，数据库正在检索",id);  
        //业务处理  
        Person person = new Person(1L,"哈哈","aa@qq.com",18,"admin");  
        //构造响应  
        return ServerResponse  
                .ok()  
                .body(person);  
    }  
  
  
    /**  
     * 获取所有用户  
     * @param request  
     * @return  
     * @throws Exception  
     */    public ServerResponse getUsers(ServerRequest request) throws Exception{  
        log.info("查询所有用户信息完成");  
        //业务处理  
        List<Person> list = Arrays.asList(new Person(1L, "哈哈", "aa@qq.com", 18, "admin"),  
                new Person(2L, "哈哈2", "aa2@qq.com", 12, "admin2"));  
  
        //构造响应  
        return ServerResponse  
                .ok()  
                .body(list); //凡是body中的对象，就是以前@ResponseBody原理。利用HttpMessageConverter 写出为json  
    }  
  
  
    /**  
     * 保存用户  
     * @param request  
     * @return  
     */  
    public ServerResponse saveUser(ServerRequest request) throws ServletException, IOException {  
        //提取请求体  
        Person body = request.body(Person.class);  
        log.info("保存用户信息：{}",body);  
        return ServerResponse.ok().build();  
    }  
  
    /**  
     * 更新用户  
     * @param request  
     * @return  
     */  
    public ServerResponse updateUser(ServerRequest request) throws ServletException, IOException {  
        Person body = request.body(Person.class);  
        log.info("保存用户信息更新: {}",body);  
        return ServerResponse.ok().build();  
    }  
  
    /**  
     * 删除用户  
     * @param request  
     * @return  
     */  
    public ServerResponse deleteUser(ServerRequest request) {  
        String id = request.pathVariable("id");  
        log.info("删除【{}】用户信息",id);  
        return ServerResponse.ok().build();  
    }  
}
```
# 测试
![[Pasted image 20240129125135.png]]
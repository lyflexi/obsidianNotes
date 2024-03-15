# Servlet是不安全的
案例一:MyServlet是单例的，内部成员变量可能会不安全
```java
public class MyServlet extends HttpServlet {  
    // 不安全  
    Map<String,Object> map = new HashMap<>();  
    // String是不可变类 ，即使修改String也只是new出新的String返回，因此String是安全的
    String S1 = "...";  
    // 安全
    final String S2 = "...";  
    // 不安全  
    Date D1 = new Date();  
    // final只是意味着引用不可变，但是Data里的内容还是可以被修改，因此final Date是不安全  
    final Date D2 = new Date();  
  
    public void doGet(HttpServletRequest request, HttpServletResponse response) {  
        // 使用上述变量  
    }  
}
```
# Service是不安全的
案例二:UserService是单例的，内部成员变量count也是单例的所以不安全
```java
public class MyServlet extends HttpServlet {  
    // 是否安全？  
    private UserService userService = new UserServiceImpl();  
  
    public void doGet(HttpServletRequest request, HttpServletResponse response) {  
        userService.update(...);  
    }  
}  
public class UserServiceImpl implements UserService {  
    // 记录调用次数  
    private int count = 0;  
  
    public void update() {  
        // ...  
        count++;  
    }  
}
```
# Aspect是不安全的
案例三:切面中的成员变量start也是不安全的，因为MyAspect是单例的start会被共享，解决方案是使用环绕通知将成员变量做成局部变量
```java
@Aspect  
@Component  
public class MyAspect {  
    // 是否安全？  
    private long start = 0L;  
  
    @Before("execution(* *(..))")  
    public void before() {  
        start = System.nanoTime();  
    }  
  
    @After("execution(* *(..))")  
    public void after() {  
        long end = System.nanoTime();  
        System.out.println("cost time:" + (end-start));  
    }  
}
```
# Dao是不安全的
案例四：UserDaoImpl中的成员变量conn是不安全的
```java
public class MyServlet extends HttpServlet {  
    // 是否安全  
    private UserService userService = new UserServiceImpl();  
  
    public void doGet(HttpServletRequest request, HttpServletResponse response) {  
        userService.update(...);  
    }  
}  
public class UserServiceImpl implements UserService {  
    // 是否安全  
    private UserDao userDao = new UserDaoImpl();  
  
    public void update() {  
        userDao.update();  
    }  
}  
public class UserDaoImpl implements UserDao {  
    // conn是不安全的  
    private Connection conn = null;  
    public void update() throws SQLException {  
        String sql = "update user set password = ? where username = ?";  
        conn = DriverManager.getConnection("","","");  
        // ...  
        conn.close();  //可能会出现误关闭，类似于分布式锁的误删问题
    }  
}
```
安全的做法是将conn做成局部变量
```java
public class UserDaoImpl implements UserDao {  
    public void update() {  
        String sql = "update user set password = ? where username = ?";  
        // 是否安全  
        try (Connection conn = DriverManager.getConnection("","","")){  
            // ...  
        } catch (Exception e) {  
            // ...  
        }  
    }  
}
```

Stream API可以极大提高Java程序员的生产力，让程序员写出高效率、干净、简洁的代码，

这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如筛选， 排序，聚合等
![[Pasted image 20231225222624.png]]

- **数据源**：流的来源可以是集合，数组，I/O channel， 产生器generator 等。
    
- **Pipelining**：中间操作都会返回流对象本身， 这样多个操作可以串联成一个管道，如同流式风格
    

# 流的生成方式

- Collection的无参实例方法：`list.stream()`和`listparallelStream()`
    
- Arrays类的静态方法：`Arrays.stream(T[])`
    
- Stream类的静态方法：`Stream.of(1, 2, 3)`
    

# Stream的中间操作函数

测试方法

```Java
@Data
public class User {
    //姓名
    private String name;
    //年龄
    private Integer age;
    //性别
    private Integer sex;
    //地址
    private String address;
    //赏金
    private BigDecimal money;

    public User(String name, Integer age, Integer sex, String address,BigDecimal money) {
        this.name = name;
        this.age = age;
        this.sex = sex;
        this.address = address;
        this.money = money;
    }
}

public class StreamTest {
    List<User> list = new ArrayList<>();
    @Before
    public void initData(){
        list = Arrays.asList(
            new User("李星云", 18, 0, "渝州",new BigDecimal(1000)),
            new User("陆林轩", 16, 1, "渝州",new BigDecimal(500)),
            new User("姬如雪", 17, 1, "幻音坊",new BigDecimal(800)),
            new User("袁天罡", 38, 0, "藏兵谷",new BigDecimal(10000)),
            new User("张子凡", 19, 0, "天师府",new BigDecimal(900)),
            new User("陆佑劫", 45, 0, "不良人",new BigDecimal(600)),
            new User("张天师", 48, 0, "天师府",new BigDecimal(1100)),
            new User("蚩梦", 18, 1, "万毒窟",new BigDecimal(800))
        );

    }


    /*filter过滤(T-> boolean)*/
    /*    ---结果---
    袁天罡 --> 99
    陆佑劫 --> 45
    张天师 --> 48*/
    @Test
    public  void filter(){
        List<User> newlist = list.stream().filter(user -> user.getAge() > 10).collect(Collectors.toList());
        for (User user : newlist) {
            System.out.println(user.getName()+" --> "+ user.getAge());
        }
    }


    /*distinct 去重  数据源中复制new User("李星云", 18, 0, "渝州",new BigDecimal(1000)) 并粘贴两个*/
    /*---结果---
    李星云 --> 18
    陆林轩 --> 16
    姬如雪 --> 17
    袁天罡 --> 99
    张子凡 --> 19
    陆佑劫 --> 45
    张天师 --> 48
    蚩梦 --> 18*/
    @Test
    public  void distinct(){
        List<User> newlist = list.stream().distinct().collect(Collectors.toList());
        for (User user : newlist) {
            System.out.println(user.getName()+" --> "+ user.getAge());
        }
    }


    /*sorted排序*/
    /*    ---结果---
    陆林轩 --> 16
    姬如雪 --> 17
    李星云 --> 18
    蚩梦 --> 18
    张子凡 --> 19
    陆佑劫 --> 45
    张天师 --> 48
    袁天罡 --> 99*/
    @Test
    public  void sorted(){
        List<User> newlist = list.stream().sorted(Comparator.comparingInt(User::getAge)).collect(Collectors.toList());
        for (User user : newlist) {
            System.out.println(user.getName()+" --> "+ user.getAge());
        }
    }


    /*limit返回前n个元素*/
    /*    ---结果---
    陆林轩 --> 16
    姬如雪 --> 17*/
    @Test
    public  void limit(){
        List<User> newlist = list.stream().sorted(Comparator.comparingInt(User::getAge)).limit(2).collect(Collectors.toList());
        for (User user : newlist) {
            System.out.println(user.getName()+" --> "+ user.getAge());
        }
    }


    /*skip去除前n个元素*/
    /*    ---结果---
    李星云 --> 18
    蚩梦 --> 18
    张子凡 --> 19
    陆佑劫 --> 45
    张天师 --> 48
    袁天罡 --> 99*/
    @Test
    public  void skip(){
        List<User> newlist = list.stream().sorted(Comparator.comparingInt(User::getAge)).skip(2).collect(Collectors.toList());
        for (User user : newlist) {
            System.out.println(user.getName()+" --> "+ user.getAge());
        }
    }


    /*map(T->R)*/
        /*    ---结果---
        李星云
        陆林轩
        姬如雪
        袁天罡
        张子凡
        陆佑劫
        张天师
        蚩梦*/
    @Test
    public  void map(){
        List<String> newlist = list.stream().map(User::getName).distinct().collect(Collectors.toList());
        for (String add : newlist) {
                System.out.println(add);
            }
    }


    /*flatMap(T -> Stream<R>)*/
    /*    ---结果---
    常宣灵
    常昊灵
    孟婆
    判官红
    判官蓝*/
    public static void flatmap(){
        List<String> flatmap = new ArrayList<>();
        flatmap.add("常宣灵,常昊灵");
        flatmap.add("孟婆,判官红,判官蓝");
        /*
        这里原集合中的数据由逗号分割，使用split进行拆分后，得到的是Stream<String[]>，
        字符串数组组成的流，要使用flatMap的Arrays::stream将Stream<String[]>转为Stream<String>,然后把流相连接
        */
        flatmap = flatmap.stream().map(s -> s.split(",")).flatMap(Arrays::stream).collect(Collectors.toList());
        for (String name : flatmap) {
                System.out.println(name);
        }
    }


    /*allMatch（T->boolean）检测是否全部满足参数行为*/
    /*        ---结果---
    false*/
    @Test
    public  void allMatch(){
        boolean flag = list.stream().allMatch(user -> user.getAge() >= 17);
        System.out.println(flag);
    }


    /*anyMatch（T->boolean）检测是否有任意元素满足给定的条件*/
    /*        ---结果---
    true*/
    @Test
    public  void anyMatch(){
        boolean flag = list.stream().anyMatch(user -> user.getSex() == 1);
        System.out.println(flag);
    }


    /*noneMatchT->boolean）流中是否有元素匹配给定的 T -> boolean条件*/
    /*        ---结果---
    true*/
    @Test
    public  void noneMatch(){
        boolean flag = list.stream().noneMatch(user -> user.getAddress().contains("郑州"));
        System.out.println(flag);
    }


    /*findFirst( ):找到第一个元素*/
    /*        ---结果---
    Optional[User{name='陆林轩', age=16, sex=1, money=500, address='渝州'}]*/
    @Test
    public  void findfirst(){
        Optional<User> optionalUser = list.stream().sorted(Comparator.comparingInt(User::getAge)).findFirst();
        System.out.println(optionalUser.toString());
    }



    /*findAny( ):找到任意一个元素*/
    /*   ---结果---
    Optional[User{name='李星云', age=18, sex=0, money=1000, address='渝州'}]*/
    @Test
    public  void findAny(){
        Optional<User> optionalUser = list.stream().findAny();
        System.out.println(optionalUser.toString());
    }



    /*计算总数*/
    /*    ---结果---
    8*/
    @Test
    public  void count(){
        long count = list.stream().count();
        System.out.println(count);
    }


    /*最大值最小值*/
    /*    max--> Optional[User{name='袁天罡', age=99, sex=0, money=100000, address='藏兵谷'}]  min--> Optional[User{name='陆林轩', age=16, sex=1, money=500, address='渝州'}]
    */
    @Test
    public  void max_min(){
        Optional<User> max = list.stream().collect(Collectors.maxBy(Comparator.comparing(User::getAge)));
        Optional<User> min = list.stream().collect(Collectors.minBy(Comparator.comparing(User::getAge)));
        System.out.println("max--> " + max+"  min--> "+ min);
    }

    /*求和_平均值*/
    /*   ---结果---
    totalAge--> 280
    totalMpney--> 105700
    avgAge--> 35.0*/
    @Test
    public  void sum_avg(){
        int totalAge = list.stream().collect(Collectors.summingInt(User::getAge));
        System.out.println("totalAge--> "+ totalAge);
    
        /*获得列表对象金额， 使用reduce聚合函数,实现累加器*/
        BigDecimal totalMpney = list.stream().map(User::getMoney).reduce(BigDecimal.ZERO, BigDecimal::add);
        System.out.println("totalMpney--> " + totalMpney);
    
        double avgAge = list.stream().collect(Collectors.averagingInt(User::getAge));
        System.out.println("avgAge--> " + avgAge);
    }


    /*一次性得到元素的个数、总和、最大值、最小值*/
    /*   ---结果---
    IntSummaryStatistics{count=8, sum=280, min=16, average=35.000000, max=99}*/
    @Test
    public  void allVlaue(){
        IntSummaryStatistics statistics = list.stream().collect(Collectors.summarizingInt(User::getAge));
        System.out.println(statistics);
    }


    /*拼接*/
    /*   ---结果---
    李星云, 陆林轩, 姬如雪, 袁天罡, 张子凡, 陆佑劫, 张天师, 蚩梦*/
    @Test
    public  void join(){
        String names = list.stream().map(User::getName).collect(Collectors.joining(", "));
        System.out.println(names);
    }


    /*分组*/
    /*   ---结果---
    {"0":[{"name":"李星云","age":18,"sex":0,"address":"渝州","money":1000},{"name":"袁天罡","age":99,"sex":0,"address":"藏兵谷","money":100000},{"name":"张子凡","age":19,"sex":0,"address":"天师府","money":900},{"name":"陆佑劫","age":45,"sex":0,"address":"不良人","money":600},{"name":"张天师","age":48,"sex":0,"address":"天师府","money":1100}],"1":[{"name":"陆林轩","age":16,"sex":1,"address":"渝州","money":500},{"name":"姬如雪","age":17,"sex":1,"address":"幻音坊","money":800},{"name":"蚩梦","age":18,"sex":1,"address":"万毒窟","money":800}]}

    {"0":{"48":[{"name":"张天师","age":48,"sex":0,"address":"天师府","money":1100}],"18":[{"name":"李星云","age":18,"sex":0,"address":"渝州","money":1000}],"19":[{"name":"张子凡","age":19,"sex":0,"address":"天师府","money":900}],"99":[{"name":"袁天罡","age":99,"sex":0,"address":"藏兵谷","money":100000}],"45":[{"name":"陆佑劫","age":45,"sex":0,"address":"不良人","money":600}]},"1":{"16":[{"name":"陆林轩","age":16,"sex":1,"address":"渝州","money":500}],"17":[{"name":"姬如雪","age":17,"sex":1,"address":"幻音坊","money":800}],"18":[{"name":"蚩梦","age":18,"sex":1,"address":"万毒窟","money":800}]}}
    */
    @Test
    public  void group(){
        Map<Integer, List<User>> map = list.stream().collect(Collectors.groupingBy(User::getSex));
        System.out.println(map.toString());
        System.out.println();
        Map<Integer, Map<Integer,List<User>>> map2 = list.stream().collect(Collectors.groupingBy(User::getSex,Collectors.groupingBy(User::getAge)));
        System.out.println(map2.toString());
    }

    /*分组合计*/
    /*   ---结果---
    {0=5, 1=3}
    {0=5, 1=1}*/
    @Test
    public  void groupCount(){
        Map<Integer, Long> num = list.stream().collect(Collectors.groupingBy(User::getSex, Collectors.counting()));
        System.out.println(num);
        Map<Integer, Long> num2 = list.stream().filter(user -> user.getAge()>=18).collect(Collectors.groupingBy(User::getSex, Collectors.counting()));
        System.out.println(num2);
    }

    /*分区*/
    /*   ---结果---
    {"false":[{"name":"袁天罡","age":99,"sex":0,"address":"藏兵谷","money":100000},{"name":"陆佑劫","age":45,"sex":0,"address":"不良人","money":600},{"name":"张天师","age":48,"sex":0,"address":"天师府","money":1100}],"true":[{"name":"李星云","age":18,"sex":0,"address":"渝州","money":1000},{"name":"陆林轩","age":16,"sex":1,"address":"渝州","money":500},{"name":"姬如雪","age":17,"sex":1,"address":"幻音坊","money":800},{"name":"张子凡","age":19,"sex":0,"address":"天师府","money":900},{"name":"蚩梦","age":18,"sex":1,"address":"万毒窟","money":800}]}
    */
    @Test
    public  void partitioningBy(){
        Map<Boolean, List<User>> part = list.stream().collect(Collectors.partitioningBy(user -> user.getAge() <= 30));
        System.out.println(part.toString());
    }
}
```
# CompletableFuture启动异步任务

## CompletableFuture创建方式：

- runAsync方法不支持返回值。其中Executor指的是可以传入我们的线程池对象
    
- supplyAsync支持返回值。其中Executor指的是可以传入我们的线程池对象
    

```Java

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the {@link ForkJoinPool#commonPool()} after
     * it runs the given action.
     *
     * @param runnable the action to run before completing the
     * returned CompletableFuture
     * @return the new CompletableFuture
     */
    public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the given executor after it runs the given
     * action.
     *
     * @param runnable the action to run before completing the
     * returned CompletableFuture
     * @param executor the executor to use for asynchronous execution
     * @return the new CompletableFuture
     */
    public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor) {
        return asyncRunStage(screenExecutor(executor), runnable);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the {@link ForkJoinPool#commonPool()} with
     * the value obtained by calling the given Supplier.
     *
     * @param supplier a function returning the value to be used
     * to complete the returned CompletableFuture
     * @param <U> the function's return type
     * @return the new CompletableFuture
     */
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }

    /**
     * Returns a new CompletableFuture that is asynchronously completed
     * by a task running in the given executor with the value obtained
     * by calling the given Supplier.
     *
     * @param supplier a function returning the value to be used
     * to complete the returned CompletableFuture
     * @param executor the executor to use for asynchronous execution
     * @param <U> the function's return type
     * @return the new CompletableFuture
     */
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor) {
        return asyncSupplyStage(screenExecutor(executor), supplier);
    }
```

## CompletableFuture回调方法：

### whenComplete

```Java
    public CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(null, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(asyncPool, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action, Executor executor) {
        return uniWhenCompleteStage(screenExecutor(executor), action);
    }
```

whenComplete和whenCompleteAsync的区别：

- whenComplete：是当前线程执行当前任务，等待任务执行之后继续执行当前的whenComplete
    
- whenCompleteAsync：是执行把whenCompleteAsync这个任务提交给线程池中的其他线程来进行执行。
    

方法以Async结尾可能会使用其他线程执行

```Java
package com.bilibili.juc.cf;

import java.util.concurrent.*;

/**
 * @auther zzyy
 * @create 2022-01-16 16:53
 */
public class CompletableFutureUseDemo
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {

        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        try
        {
            CompletableFuture.supplyAsync(() -> {
                System.out.println(Thread.currentThread().getName() + "----come in");
                int result = ThreadLocalRandom.current().nextInt(10);
                try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println("-----1秒钟后出结果：" + result);
                if(result > 2)
                {
                    int i=10/0;
                }
                return result;
            },threadPool).whenComplete((v,e) -> {
                if (e == null) {
                    System.out.println("-----计算完成，更新系统UpdateValue："+v);
                }
            }).exceptionally(e -> {
                e.printStackTrace();
                System.out.println("异常情况："+e.getCause()+"\t"+e.getMessage());
                return null;
            });

            System.out.println(Thread.currentThread().getName()+"线程先去忙其它任务");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            threadPool.shutdown();
        }


        //主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:暂停3秒钟线程
        //try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }

    }

    private static void future1() throws InterruptedException, ExecutionException
    {
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "----come in");
            int result = ThreadLocalRandom.current().nextInt(10);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("-----1秒钟后出结果：" + result);
            return result;
        });

        System.out.println(Thread.currentThread().getName()+"线程先去忙其它任务");

        System.out.println(completableFuture.get());
    }
}
```

### exceptionally

exceptionally处理异常情况

```Java
    /**
     * Returns a new CompletableFuture that is completed when this
     * CompletableFuture completes, with the result of the given
     * function of the exception triggering this CompletableFuture's
     * completion when it completes exceptionally; otherwise, if this
     * CompletableFuture completes normally, then the returned
     * CompletableFuture also completes normally with the same value.
     * Note: More flexible versions of this functionality are
     * available using methods {@code whenComplete} and {@code handle}.
     *
     * @param fn the function to use to compute the value of the
     * returned CompletableFuture if this CompletableFuture completed
     * exceptionally
     * @return the new CompletableFuture
     */
    public CompletableFuture<T> exceptionally(
        Function<Throwable, ? extends T> fn) {
        return uniExceptionallyStage(fn);
    }
```

handle

handle：whenComplete和exceptionally的结合版。方法执行后的处理，无论成功与失败都可处理

```Java
    public <U> CompletableFuture<U> handle(
        BiFunction<? super T, Throwable, ? extends U> fn) {
        return uniHandleStage(null, fn);
    }

    public <U> CompletableFuture<U> handleAsync(
        BiFunction<? super T, Throwable, ? extends U> fn) {
        return uniHandleStage(asyncPool, fn);
    }

    public <U> CompletableFuture<U> handleAsync(
        BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) {
        return uniHandleStage(screenExecutor(executor), fn);
    }
```

代码示例：

```Java
                /**
                 * 方法执行完成后的处理
                 */
        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("当前线程：" + Thread.currentThread().getId());
            System.out.println("CompletableFuture...");
            return 10/1;
        }, service).handle((t,u)->{ // R apply(T t, U u);
            System.out.println("handle:");
            if (t != null){
                System.out.println("存在返回结果:" + t);
                return 8;
            }
            if (u != null){
                System.out.println("存在日常:" + u);
                return 9;
            }
            return 5;

        });
        Integer integer = completableFuture2.get();
        System.out.println(integer);
```

# CompletableFuture定制化场景

## 线程串行化
![[Pasted image 20231225134120.png]]

- thenRun：不能获取到上一步的执行结果，无返回值
    
- thenAcceptAsync 能接受上—步结果，但是无返回值
    
- **thenApplyAsync** 能接受上—步结果，有返回值
    

**我们即要能感知到上一步的执行结果，也要能有自己线程执行成功的自己的返回值**

```Java
我是B
//        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
//            System.out.println("当前线程：" + Thread.currentThread().getId());
//            System.out.println("CompletableFuture...");
//            return 10;//拿到A的返回值
//        }, service).thenApplyAsync((u)->{
//            System.out.println("返回值" + u);
//            System.out.println("任务2启动");
//            return 5;//自己的返回值再返回出去
//        });
//        System.out.println(completableFuture2.get());
        /*
        * main....start....
        当前线程：11
        CompletableFuture...
        返回值10
        任务2启动
        5
        main....end....
                * */
```

## 双线程均完成才能后续

![[Pasted image 20231225134135.png]]
![[Pasted image 20231225134141.png]]

- runAfterBothAsync 两人任务组合，不能得到前任务的结果和无返回值
    
- thenAcceptBothAsync 两人任务组合，能得到前任务的结果和无返回值
    
- thenCombineAsync 两人任务组合，能得到前任务的结果和有返回值
    

代码示例

```Java
//        CompletableFuture<Integer> completableFuture3 = CompletableFuture.supplyAsync(() -> {
//                    System.out.println("当前线程：" + Thread.currentThread().getId());
//                    System.out.println("任务1...");
//                    return 111;
//                }, service);
//        CompletableFuture<Integer> completableFuture4 = CompletableFuture.supplyAsync(() -> {
//            System.out.println("当前线程：" + Thread.currentThread().getId());
//            System.out.println("任务2...");
//            return 222;
//        }, service);



//        completableFuture3.runAfterBothAsync(completableFuture4,()->{
//                System.out.println("任务3...");
//        },service);
        /*
        * main....start....
        main....end....
        当前线程：11
        任务1...
        当前线程：12
        任务2...
        任务3...
        * */

//        completableFuture3.thenAcceptBothAsync(completableFuture4, (f1,f2) -> {
//            System.out.println("任务3...");
//            System.out.println("f1:" + f1 + ".f2:" + f2);
//        }, service);
        /*
        * main....start....
        main....end....
        当前线程：11
        任务1...
        当前线程：12
        任务2...
        任务3...
        f1:111.f2:222
        * */

//        CompletableFuture<Integer> integerCompletableFuture = completableFuture3.thenCombineAsync(completableFuture4, (f1, f2) -> {
//            System.out.println("任务3...");
//            System.out.println("f1:" + f1 + ".f2:" + f2);
//            return 3;
//        }, service);
//        System.out.println(integerCompletableFuture.get());
        /*
        * main....start....
        当前线程：11
        任务1...
        当前线程：12
        任务2...
        任务3...
        f1:111.f2:222
        3
        main....end....
        * */
```

## 双线程完成其一就能后续
![[Pasted image 20231225134207.png]]
![[Pasted image 20231225134212.png]]

runAfterEither：两个任务有一个执行完成，不需要获取future的结果，处理任务，也没有返回值。

acceptEither：两个任务有一个执行完成，获取它的返回值

applyToEither：**两个任务有一个执行完成，获取它的返回值，处理任务并有新的返回值。

```Java
//        CompletableFuture<Integer> completableFuture5 = CompletableFuture.supplyAsync(() -> {
//            System.out.println("当前线程：" + Thread.currentThread().getId());
//            System.out.println("任务1...");
//            return 111;
//        }, service);
//
//        CompletableFuture<Integer> completableFuture6 = CompletableFuture.supplyAsync(() -> {
//            System.out.println("当前线程：" + Thread.currentThread().getId());
//            try {
//                Thread.sleep(2000);
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//            }
//            System.out.println("任务2结束...");
//            return 222;
//        }, service);


//        completableFuture5.runAfterEitherAsync(completableFuture6, () -> {
//            System.out.println("任务3...");
//        }, service);
        /*
        * main....start....
        main....end....
        当前线程：11
        任务1...
        当前线程：12
        任务3...
        任务2结束...
        * */

//        completableFuture5.acceptEitherAsync(completableFuture6, (f1) -> {
//            System.out.println("f1:" + f1);
//            System.out.println("任务3...");
//        }, service);
        /*
        * main....start....
        当前线程：11
        任务1...
        当前线程：12
        main....end....
        f1:111
        任务3...
        任务2...
        * */

//        CompletableFuture<Integer> integerCompletableFuture = completableFuture5.applyToEitherAsync(completableFuture6, (f1) -> {
//            System.out.println("f1:" + f1);
//            System.out.println("任务3...");
//            return 6;
//        }, service);
//        System.out.println(integerCompletableFuture.get());
        /*
        * main....start....
        当前线程：11
        任务1...
        当前线程：12
        f1:111
        任务3...
        6
        main....end....
        任务2结束...
        * */
```

多任务组合

1、`allOf`：等待所有任务完成

2、`anyOf`：只要有一个任务完成

```Java
    CompletableFuture<String> img = CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品图片信息");
            return "1.jpg";
    },service);

    CompletableFuture<String> attr = CompletableFuture.supplyAsync(() -> {
            try {
                    Thread.sleep(2000);
            } catch (InterruptedException e) {
                    e.printStackTrace();
            }
            System.out.println("查询商品属性");
            return "麒麟990 5G  钛空银";
    },service);


    CompletableFuture<String> desc = CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品介绍");
            return "华为";
    },service);

    /**
     * 等这三个都做完
     */

    CompletableFuture<Void> allOf = CompletableFuture.allOf(img, attr, desc);
    allOf.join();

//                System.out.println("main....end"  + desc.get() + attr.get() + img.get());
//                CompletableFuture<Object> anyOf = CompletableFuture.anyOf(img, attr, desc);
//                anyOf.get();

    System.out.println("main....end" + img.get()+attr.get()+desc.get());

main....start
查询商品图片信息
查询商品介绍
#这里卡2s
查询商品属性
main....end1.jpg麒麟990 5G  钛空银华为
```

# CompletableFuture+自定义线程池

## MyThreadConfig

线程池配置

```Java
/**
 * @Description:  本源码分享自 www.cx1314.cn   欢迎访问获取更多资源 线程池配置类
 * @Created: 程序源码论坛
 * @author: cx
 * @createTime: 2020-06-23 20:24
 **/

@EnableConfigurationProperties(ThreadPoolConfigProperties.class)
@Configuration
public class MyThreadConfig {


    @Bean
    public ThreadPoolExecutor threadPoolExecutor(ThreadPoolConfigProperties pool) {
        return new ThreadPoolExecutor(
                pool.getCoreSize(),
                pool.getMaxSize(),
                pool.getKeepAliveTime(),
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(100000),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()
        );
    }

}
```

## ThreadPoolConfigProperties

将配置参数抽取出来到application.properties

加上@Component，或者MyThreadConfig导入ThreadPoolConfigProperties.class的时候加上@EnableConfigurationProperties注解即可

```Java
@ConfigurationProperties(prefix = "gulimall.thread")
// @Component
@Data
public class ThreadPoolConfigProperties {

    private Integer coreSize;

    private Integer maxSize;

    private Integer keepAliveTime;


}
```

## 业务实现

```Java
    @Override
    public SkuItemVo item(Long skuId) throws ExecutionException, InterruptedException {

        SkuItemVo skuItemVo = new SkuItemVo();

        CompletableFuture<SkuInfoEntity> infoFuture = CompletableFuture.supplyAsync(() -> {
            //1、sku基本信息的获取  pms_sku_info
            SkuInfoEntity info = this.getById(skuId);
            skuItemVo.setInfo(info);
            return info;
        }, executor);


        CompletableFuture<Void> saleAttrFuture = infoFuture.thenAcceptAsync((res) -> {
            //3、获取spu的销售属性组合
            List<SkuItemSaleAttrVo> saleAttrVos = skuSaleAttrValueService.getSaleAttrBySpuId(res.getSpuId());
            skuItemVo.setSaleAttr(saleAttrVos);
        }, executor);


        CompletableFuture<Void> descFuture = infoFuture.thenAcceptAsync((res) -> {
            //4、获取spu的介绍    pms_spu_info_desc
            SpuInfoDescEntity spuInfoDescEntity = spuInfoDescService.getById(res.getSpuId());
            skuItemVo.setDesc(spuInfoDescEntity);
        }, executor);


        CompletableFuture<Void> baseAttrFuture = infoFuture.thenAcceptAsync((res) -> {
            //5、获取spu的规格参数信息
            List<SpuItemAttrGroupVo> attrGroupVos = attrGroupService.getAttrGroupWithAttrsBySpuId(
                res.getSpuId(), res.getCatalogId());
            skuItemVo.setGroupAttrs(attrGroupVos);
        }, executor);


        // Long spuId = info.getSpuId();
        // Long catalogId = info.getCatalogId();

        //2、sku的图片信息    pms_sku_images
        CompletableFuture<Void> imageFuture = CompletableFuture.runAsync(() -> {
            List<SkuImagesEntity> imagesEntities = skuImagesService.getImagesBySkuId(skuId);
            skuItemVo.setImages(imagesEntities);
        }, executor);

        CompletableFuture<Void> seckillFuture = CompletableFuture.runAsync(() -> {
            //3、远程调用查询当前sku是否参与秒杀优惠活动
            R skuSeckilInfo = seckillFeignService.getSkuSeckilInfo(skuId);
            if (skuSeckilInfo.getCode() == 0) {
                //查询成功
                SeckillSkuVo seckilInfoData = skuSeckilInfo.getData("data", new TypeReference<SeckillSkuVo>() {
                });
                skuItemVo.setSeckillSkuVo(seckilInfoData);

                if (seckilInfoData != null) {
                    long currentTime = System.currentTimeMillis();
                    if (currentTime > seckilInfoData.getEndTime()) {
                        skuItemVo.setSeckillSkuVo(null);
                    }
                }
            }
        }, executor);


        //等到所有任务都完成
        CompletableFuture.allOf(saleAttrFuture,descFuture,baseAttrFuture,imageFuture,seckillFuture).get();

        return skuItemVo;
    }
```

## application.properties如下

```Java
spring.cache.type=redis

#spring.cache.cache-names=qq,毫秒为单位
spring.cache.redis.time-to-live=3600000

#如果指定了前缀就用我们指定的前缀，如果没有就默认使用缓存的名字作为前缀
#spring.cache.redis.key-prefix=CACHE_
spring.cache.redis.use-key-prefix=true

#是否缓存空值，防止缓存穿透
spring.cache.redis.cache-null-values=true

#配置线程池
gulimall.thread.coreSize=20
gulimall.thread.maxSize=200
gulimall.thread.keepAliveTime=10

#开启debug日志
logging.level.org.springframework.cloud.openfeign=debug
logging.level.org.springframework.cloud.sleuth=debug

#服务追踪
spring.zipkin.base-url=http://192.168.18.80:9411/
#关闭服务发现
spring.zipkin.discovery-client-enabled=false
spring.zipkin.sender.type=web
#配置采样器
spring.sleuth.sampler.probability=1
```
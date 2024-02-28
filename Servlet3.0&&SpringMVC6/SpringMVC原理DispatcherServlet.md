SpringMVC是对Servlet的封装，屏蔽掉Servlet很多的细节，举几个例子

> 可能我们刚学Servlet的时候，要获取参数需要不断的getParameter
> 现在只要在SpringMVC方法定义对应的JavaBean，只要属性名与参数名一致，SpringMVC就可以帮我们实现「将参数封装到JavaBean」上了
> 
> 又比如，以前使用Servlet「上传文件」，需要处理各种细节，写一大堆处理的逻辑（还得导入对应的jar）
> 现在一个在SpringMVC的方法上定义出MultipartFile接口，又可以屏蔽掉上传文件的细节了。

![[Pasted image 20240108160808.png]]

# SpringMVC初始化阶段

==SpringMVC的初始化使用了大量的模板设计模式：==
## HttpServlet

### HttpServletBean

抽象类HttpServletBean继承了HttpServlet，重写了`init`方法，最终调用`initServletBean`方法，

但是HttpServletBean自己本身并没有实现`initServletBean`方法，而是把`initServletBean`方法留给了子类`FrameworkServlet`去实现，以及由子类`FrameworkServlet`调用`initServletBean`方法
![[Pasted image 20240108160900.png]]

#### FrameworkServlet初始化父子容器WAC
![[Pasted image 20240108160933.png]]

核心方法initWebApplicationContext方法如下：
![[Pasted image 20240108161021.png]]

在`configureAndRefreshWebApplicationContext`方法中：

1. 建立了父子容器的关联关系。
    
2. 并且最终调用了`AbstractApplicationContext`的`refresh()`方法来初始化IOC容器。

![[Pasted image 20240108161052.png]]

`configureAndRefreshWebApplicationContext(wac)`方法传入的是一个`ConfigurableWebApplicationContext`接口，但是`AbstractApplicationContext`实现了`ConfigurableWebApplicationContext`接口。

![[Pasted image 20240108161124.png]]

##### DispatcherServlet初始化SpringMVC九大组件onRefresh
![[Pasted image 20240108161146.png]]

- HandlerMapping：`HandlerMapping`是用来查找`Handler`的，也就是处理器，具体表现形式可以是类也 可以是方法。比如，标注了`@RequestMapping`的每个`method`都可以看成是一个`Handler`，由`Handler`来负责实际的请求处理。`HandlerMapping`在请求到达之后， 它的作用便是找到请求相应的处理器`Handler`，==同时找到拦截器`Interceptors`==。
    
- HandlerAdapter：从名字上看，这是一个适配器。因为Spring MVC中`Handler`可以是任意形式的，只要能够处理请求就行。但是把请求交给 `Servlet`的时候，由于`Servlet`的方法结构都是如`doService(HttpServletRequest req, HttpServletResponse resp)`这样的形式，让固定的`Servlet`处理方法调用`Handler`来进行处理，这一步工作便是`HandlerAdapter`要做的事。
    
- HandlerExceptionResolver：从这个组件的名字上看，这个就是用来处理`Handler`过程中产生的异常情况的组件。 具体来说，此组件的作用是根据异常设置 `ModelAndView`, 之后再交给`render()`方法进行渲染，而`render()`便将`ModelAndView`渲染成页面。 不过有一点，`HandlerExceptionResolver`只是用于解析对请求做处理阶段产生的异常，而渲染阶段的异常则不归他管了，这也是Spring MVC组件设计的一大原则（分工明确互不干涉）。
    
- ViewResolver：视图解析器，相信大家对这个应该都很熟悉了。因为通常在Spring MVC的配置文件中， 都会配上一个该接口的实现类来进行视图的解析。 这个组件的主要作用，便是将`String`类型的视图名和`Locale`解析为`View`类型的视图。这个接口只有一个 `resolveViewName()`方法。从方法的定义就可以看出，`Controller`层返回的`String`类型的视图名`viewName`， 最终会在这里被解析成为`View`。`View`是用来渲染页面的，也就是说，它会将程序返回的参数和数据填入模板中，最终生成`html` 文件。`ViewResolver`在这个过程中，主要做两件大事，即`ViewResolver`会找到渲染所用的模板(使用什么模板来渲染?)和所用 的技术(其实也就是视图的类型，如 JSP等)填入参数。默认情 况下，Spring MVC 会为我们自动配置一个 `InternalResourceViewResolver`，这个是针 对 JSP 类型视图的。
    
- RequestToViewNameTranslator：这个组件的作用，在于从`Request`中获取`viewName`. 因为`ViewResolver`是根据`ViewName`查找`View`, 但有的 `Handler`处理完成之后，没有设置`View`也没有设置`ViewName`， 便要通过这个组件来从`Request`中查找`viewName`。
    
- LocaleResolver：在上面我们有看到`ViewResolver`的`resolveViewName()`方法，需要两个参数。那么第二个参数`Locale`是从哪来的呢，这就是`LocaleResolver`要做的事了。 `LocaleResolver`用于从`request`中解析出`Locale`, 在中国大陆地区，`Locale`当然就会是`zh-CN`之类， 用来表示一个区域。这个类也是`i18n`的基础。
    
- ThemeResolver：从名字便可看出，这个类是用来解析主题的。主题就是样式，图片以及它们所形成的显示效果的集合。
    
- MultipartResolver：其实这是一个大家很熟悉的组件，`MultipartResolver`用于处理上传请求，通过将普通的`Request`包装成 `MultipartHttpServletRequest`来实现。`MultipartHttpServletRequest`可以通过`getFile()`直接获得文件，如果是多个文件上传，还可以通过调用`getFileMap`得到`Map`这样的结构。`MultipartResolver`的作用就是用来封装普通 的`request`，使其拥有处理文件上传的功能。
    
- FlashMapManager：`FlashMap`用于重定向`Redirect`时的参数数据传递, 只需要在`redirect`之前，将要传递的数据写入`request`(可以通过 `ServletRequestAttributes.getRequest()` 获得) 的属性`OUTPUT_FLASH_MAP_ATTRIBUTE`中，这样在 `redirect`之后的`handler`中，Spring就会自动将其设置到`Model`中，后续就可以直接从`Model`中取得数据了。而 `FlashMapManager`就是用来管理`FlashMap`的。
    

# DispatcherServlet的执行阶段

**SpringMVC是对Servlet的封装，SpringMVC请求处理的流程如下**

1. 首先有个统一处理请求的入口`DispatcherServlet`（前端控制器）：通过DispatcherServlet.properties文件配置，会把映射器、适配器、视图解析器、异常处理器、文件处理器等等给初始化掉

2. 随后根据请求路径找到对应的`HanlderMapping`映射器，映射器`HanlderMapping`同时也是个HandlerExecutionChain拦截器List
3. 对处理映射器`HanlderMapping`进行包装，得到`HandlerAdapter`（适配器）
4. 拦截器前置处理
5. 真实处理请求调用真正的代码->利用第3步的适配器，调用包装后的`Handler`中的方法，处理业务
    1. 注解请求参数绑定方法参数
    2. 返回结果通过Converter转换为JSON
    3. 处理结束，返回ModelAndView（Model模型数据+View视图名称（不是真正的视图））
    4. `DispatcherServlet`将视图名称交给ViewResolver（视图解析器），==ViewResolver返回真正的视图对象==给`DispatcherServlet`，`DispatcherServlet`把`Model`（数据模型）交给视图对象进行渲染，将渲染后的视图返回用户，产生响应。
6. 拦截器后置处理
    
![[Pasted image 20240108161407.png]]

统一的处理入口对应SpringMVC下的源码是在DispatcherServlet下实现的
## doService->doDispatch
所有的请求其实都会被doService方法处理，翻译过来如下：
子类必须实现doService方法来完成请求处理的工作，接收GET、POST、PUT和DELETE的集中回调。该契约本质上与HttpServlet中通常被覆盖的doGet或doPost方法的契约相同。这个类拦截调用，以确保异常处理和事件发布发生。参数:request-当前HTTP请求response -当前HTTP响应抛出:Exception-在任何类型的处理失败的情况下参见:jakarta.servlet.http.HttpServlet。doGet, jakarta.servlet.http.HttpServlet.doPost
```java
/**  
 * Subclasses must implement this method to do the work of request handling, * receiving a centralized callback for GET, POST, PUT and DELETE. * <p>The contract is essentially the same as that for the commonly overridden  
 * {@code doGet} or {@code doPost} methods of HttpServlet.  
 * <p>This class intercepts calls to ensure that exception handling and  
 * event publication takes place. * @param request current HTTP request  
 * @param response current HTTP response  
 * @throws Exception in case of any kind of processing failure  
 * @see jakarta.servlet.http.HttpServlet#doGet  
 * @see jakarta.servlet.http.HttpServlet#doPost  
 */

 protected abstract void doService(HttpServletRequest request, HttpServletResponse response)  
       throws Exception;
```

doService里边最主要就是调用doDispatch方法，通过doDispatch方法我们就可以看到整个SpringMVC处理的流程

```Java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
       ModelAndView mv = null;
       Exception dispatchException = null;

       try {
          processedRequest = checkMultipart(request);
          multipartRequestParsed = (processedRequest != request);

          // Determine handler for the current request
          //映射器HandlerExecutionChain，封装了(映射器+拦截器List)
          mappedHandler = getHandler(processedRequest);
          if (mappedHandler == null) {
             noHandlerFound(processedRequest, response);
             return;
          }

          // Determine handler adapter for the current request.
          //适配器HandlerAdapter，对映射器进行了包装
          HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

          // Process last-modified header, if supported by the handler.
          String method = request.getMethod();
          boolean isGet = HttpMethod.GET.matches(method);
          if (isGet || HttpMethod.HEAD.matches(method)) {
             long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
             if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                return;
             }
          }

          //拦截器前置处理
          if (!mappedHandler.applyPreHandle(processedRequest, response)) {
             return;
          }

          // Actually invoke the handler.
          //实际处理，返回视图对象ModelAndView 
          mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

          if (asyncManager.isConcurrentHandlingStarted()) {
             return;
          }

          applyDefaultViewName(processedRequest, mv);
          //拦截器后置处理
          mappedHandler.applyPostHandle(processedRequest, response, mv);
       }
       catch (Exception ex) {
          dispatchException = ex;
       }
       catch (Throwable err) {
          // As of 4.3, we're processing Errors thrown from handler methods as well,
          // making them available for @ExceptionHandler methods and other scenarios.
          dispatchException = new ServletException("Handler dispatch failed: " + err, err);
       }
       processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
       triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
       triggerAfterCompletion(processedRequest, response, mappedHandler,
             new ServletException("Handler processing failed: " + err, err));
    }
    finally {
       if (asyncManager.isConcurrentHandlingStarted()) {
          // Instead of postHandle and afterCompletion
          if (mappedHandler != null) {
             mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
          }
       }
       else {
          // Clean up any resources used by a multipart request.
          if (multipartRequestParsed) {
             cleanupMultipart(processedRequest);
          }
       }
    }
}
```

## 获取映射器HandlerExecutionChain

查找映射器的时候实际就是找到「最佳匹配」的路径：比如我的测试请求路径是：com.springdebug.mvc.DebugController#testRequestParam(String)

查找映射器实际返回的是HandlerExecutionChain，里边有映射器Handler+拦截器List

![[Pasted image 20240108161615.png]]
![[Pasted image 20240108161620.png]]

## 获取适配器RequestMappingHandlerAdapter

适配器是对HandlerExecutionChain的进一步封装，一般我们用的适配器是`RequestMappingHandlerAdapter`
![[Pasted image 20240108161626.png]]

## 拦截器前置处理

执行拦截器`HandlerInterceptor`的前置处理方法`preHandle()`。

## 真实请求处理（利用适配器RequestMappingHandlerAdapter）

拦截器前置处理执行完后，就会调用适配器对象实例的handle
```java
// Actually invoke the handler.
//实际处理，返回视图对象ModelAndView 
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```
跟踪代码到`RequestMappingHandlerAdapter`的`handleInternal`方法：
![[Pasted image 20240108161636.png]]

继续跟踪代码到`RequestMappingHandlerAdapter`的`invokeAndHandle`方法：
![[Pasted image 20240108161643.png]]

### 注解请求参数绑定方法参数invokeForRequest

在`invocableMethod.invokeAndHandle()`中完成了`Request`中的参数和方法参数上数据的绑定。Spring MVC中提供两种`Request`参数到方法中参数的绑定方式:

- 通过注解`@RequestParam`进行绑定。使用注解进行绑定，我们只要在方法参数前面声明`@RequestParam("name")`，就可以 将`request`中参数`name`的值绑定到方法的该参数上。
    
- 通过参数名称进行绑定，使用参数名称进行绑定的前提是必须要获取方法中参数的名称。Spring基于ASM技术实现了运行时获取参数的实现。
    
![[Pasted image 20240108161652.png]]

### 返回结果JSON转化RequestResponseBodyMethodProcessor
![[Pasted image 20240108161701.png]]

继续跟进`handleReturnValue()`方法，对于常见的`json`响应体返回，实际调用了`RequestResponseBodyMethodProcessor`的实现：

![[Pasted image 20240108161710.png]]

在执行真正的消息转换器写入之前，会先调用`ResponseBodyAdvice`的`beforeBodyWrite()`方法一些前置处理，在这里我们可以改写返回的`body`数据。

最后根据实际的消息转换器，将方法返回值写入到response中，在执行完后续操作之后，直接将结果输出。
![[Pasted image 20240108161721.png]]
## 拦截器后置处理

执行拦截器HandlerInterceptor的后置处理postHandle()
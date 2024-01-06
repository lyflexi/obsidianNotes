-  AspectJAroundAdvice
    
- MethodBeforeAdviceInterceptor
    
- AspectJAfterAdvice
    
- AfterReturningAdviceInterceptor
    
- AspectJAfterThrowingAdvice
    

上述五个拦截器都属于`MethodInterceptor`，因为他们都`implements MethodInterceptor`

# 环绕通知的执行AspectJAroundAdvice#invoke

```Java
public Object invoke(MethodInvocation mi) throws Throwable {
    if (!(mi instanceof ProxyMethodInvocation)) {
            throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
    }
    ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
    ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
    JoinPointMatch jpm = getJoinPointMatch(pmi);
    return invokeAdviceMethod(pjp, jpm, null, null);
}
```

这里会去调用环绕通知的增强方法，而环绕通知的增强方法中会执行proceedingJoinPoint.proceed()，

这样就会调用下一个MethodInterceptor–>MethodBeforeAdviceInterceptor。

# 前置通知的执行MethodBeforeAdviceInterceptor#invoke

==MethodBeforeAdviceInterceptor其实是将MethodBeforeAdvice封装了起来==。先执行Before增强（MethodBeforeAdvice），再调用MethodInvocation.proceed()传递给下一个MethodInterceptor。
![[Pasted image 20240106100834.png]]

后面的通知方法以此类推

# 后置通知的执行AspectJAfterAdvice#invoke

先执行MethodInvocation.proceed()，最后在finally块中调用后置通知的增强，不管目标方法有没有抛出异常，finally代码块中的代码都会执行，也就是不管目标方法有没有抛出异常，后置通知都会执行

```Java
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
            return mi.proceed();
    }
    finally {
            invokeAdviceMethod(getJoinPointMatch(), null, null);
    }
}
```

# 返回后通知的执行AfterReturningAdviceInterceptor#invoke

先执行MethodInvocation.proceed()，然后再执行返回后通知的增强。

```Java
public Object invoke(MethodInvocation mi) throws Throwable {
    Object retVal = mi.proceed();
    this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
    return retVal;
}
```

# 异常通知的执行AspectJAfterThrowingAdvice#invoke

先执行MethodInvocation.proceed()，如果目标方法抛出了异常就会执行异常通知的增强，然后抛出异常

```Java
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
            return mi.proceed();
    }
    catch (Throwable ex) {
            if (shouldInvokeOnThrowing(ex)) {
                    invokeAdviceMethod(getJoinPointMatch(), null, ex);
            }
            throw ex;
    }
}
```

# 整个拦截链的过程

所谓的拦截器链其实就是将每一个增强方法

1. AspectJAroundAdvice
    
2. MethodBeforeAdvice
    
3. AspectJAfterAdvice
    
4. AfterReturningAdvice
    
5. AspectJAfterThrowingAdvice
    

又被包装为了方法拦截器MethodInterceptor：

1. AspectJAroundAdviceInterceptor
    
2. MethodBeforeAdviceInterceptor
    
3. AspectJAfterAdviceInterceptor
    
4. AfterReturningAdviceInterceptor
    
5. AspectJAfterThrowingAdviceInterceptor
    

比如：前置增强方法-->前置拦截器

```Java
MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
return new MethodBeforeAdviceInterceptor(advice);
```

利用拦截器的链式机制，在每个拦截器MethodInterceptor中，依次每一个拦截器中的invoke()方法，

- 同时利用火炬MethodInvocation mi（ReflectiveMethodInvocation），进行火炬的传递mi.proceed();
    

最终，整个的执行效果就会有两套：

- 目标方法正常执行：前置通知→目标方法→后置通知→返回通知
    
- 目标方法出现异常：前置通知→目标方法→后置通知→异常通知
    
![[Pasted image 20240106100846.png]]

整个拦截器链的执行过程，我们总结一下，其实就是链式执行每一个拦截器的invoke()方法，

通过拦截器链这种机制，保证了通知方法与目标方法的执行顺序。
---
layout: post
title: "Spring AOP代理"
description: "Spring AOP代理"
categories: [spring]
tags: [aop]
redirect_from:
  - /2017/06/29/
---
# Spring AOP代理的相关属性

- 开启AOP自动代理

```
<aop:aspect-autoproxy />
```

- proxy-target-class:强调spring应该使用那种代理方式：JDK动态代理和CGLIB
  - JDK动态代理：代理对象必须为某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理【默认属性值为false，即使用的是JDK动态代理】
  - CGLIB代理：原理类似于JDK代理，不同之处在于运行期间生成的代理对象是针对目标类扩展的子类，CGLIB是高效代码生成包，底层是依靠ASM（字节码编辑类库）操作字节码实现的，性能比JDK强。
  - 使用CGLIB代理会出现的问题:无法通知Final方法，因为他们不允许被覆盖；还需要将CGLIB二进制包放入classpath下面；强制使用CGLIB代理需要将proxy-target-class设置为true
- expose-proxy:目标对象内部的自我调用将无法实施切面中的增强


# AOP代理无法切入同类调用方法的问题

```
@Service
public class Service {  
  
    //这个方法调用被切的方法,在外部被调用 
    public void callMethodA() {  
        ......  
        callMethodB();  
        ......  
    }  
      
    //Aop切入的方法 
    public void callMethodB() {  
        ......  
    }  
} 

@Aspect
@Service
public class Aspect {
    @After("execution(* Service.callMethodB(..))")  
    public void after() {  
        Logger.info("after call and do something.");
    }  
}  

//调用方式
service.callMethodA();

//spring配置
<aop:aspect-autoproxy proxy-target-class="true"/>
```

上述的调用方式是根本无法进去切面的，在callMethodA()方法中调用callMethodB()相当于调用this.callMethodB()，此处的this指向目标对象，并不是调用代理对象的callMethodB()方法，解决方式如下两点
- 强制使用AOP代理

```
//spring配置
<aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true" />

//调用callMethodB()
((Service)AopContext.currentProxy()).callMethodB();
```

- 通过spring上下文获取代理类

```
//spring配置
<aop:aspectj-autoproxy proxy-target-class="true"/>

//调用callMethodB()
applicationContext.getBean("service").callMethodB();
```

PS:使用第一种方式的时候要注意获取代理对象时的强制转换:使用默认的JDK动态代理的方式需要转换为接口，使用CGLIB代理的方式可以直接使用实现类的类型；同时代理对象使用的方法需要是public修饰的方法；spring自带的@transactional注解在使用中也有上述问题，该注解默认使用CGLIB代理。

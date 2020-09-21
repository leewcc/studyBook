# Spring AOP
AOP 是 spring 的核心特性，面向切面编程，支持将不同的功能通过 AOP 的机制横切织入到目标方法中，从而加强目标方法。

# 基本概念
## advice
通知，是指拦截到连接点时要执行的方法，常见的通知类型
- before：前置通知，方法执行前被触发
- arround：环绕通知，可环绕方法的执行过程来执行
- after：后置通知，方法执行后被触发
- afterReturn：方法执行返回通知，方法返回时触发
- afterThrow：抛出异常通知，抛出异常时通知

通知的执行顺序：
arround → before → target.process → after → afterReturn
arround → before → target.process → after → afterThrow

## pointcut
切点，定义连接点进行拦截器的条件定义，支持表达式定义

## jointpoint
连接点，被拦截到的方法
## target
被拦截的目标对象

## aspect
切面，由 pointcut、advice 组成，表示一个横切点，定义了横切点的功能模块

## advisor
切面增强器，能够将切面织入到目标对象中

# AOP 的原理
spring AOP 的核心是代理，通过将目标对象包装成代理类，通过代理将多个切面横切织入到目标方法中，这里面的主要运用到了两种设计模式——代理模式、责任链模式

在构造 bean 调用 wrapIfNecessary 时，会调用 getAdvicesAndAdvisorsForBean 方法构造拦截器链织入到目标方法中，当目标方法被调用时，则会触发拦截器链调用，根据拦截器的类型进行不同处理

**核心类AbstractAdvisorAutoProxyCreator**


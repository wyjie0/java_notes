# 什么是Spring

Spring是一种轻量级（编写更少的代码和更少的配置问价；）的Java开发框架，为了解决业务逻辑层和其他各层的耦合问题。

Spring最根本的使命是解决企业级开发的复杂性，简化Java开发。它提供的功能都依赖于两个核心：依赖注入和AOP（面向切面编程）

# 一、Spring事务中的事务传播行为

## 1、什么是事务传播行为

事务传播行为用来描述由某一个事务传播行为修饰的方法被嵌套进入另一个方法的时候事务如何传播

```java
public void methodA(){
    methodB();
    //doSomething
 }
 
 @Transaction(Propagation=XXX)
 public void methodB(){
    //doSomething
 }
```

代码中`methodA()`方法嵌套调用了`methodB()`方法，`methodB()`的事务传播行为由`@Transaction(Propagation=XXX)`设置决定。这里需要注意的是`methodA()`并没有开启事务，某一个事务传播行为修饰的方法并不是必须要在开启事务的外围方法中调用。

## 2、Spring中的7种事务传播行为

| 事务传播行为类型          | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | 如果当前没有事务，就新建一个事务；如果已经存在于一个事务当中，加入到该事务中。这是最常见的选择 |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务的方式执行         |
| PROPAGATION_MANDATORY     | 使用当前事务，如果当前没有事务，就抛出异常                   |
| PROPAGATION_REQUIRED_NEW  | 新建事务，如果当前存在事务，就把当前事务挂起                 |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起     |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常               |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作 |

* PROPAGATION_REQUIRED

  在外围方法**没有开启事务**的情况下，`PROPAGATION_REQUIRED`修饰的内部方法会新开启自己的事务，且**开启的事务相互独立，互补干扰**

  在外围方法**开启事务**的情况下，`PROPAGATION_REQUIRED`修饰的内部方法会加入到外围方法的事务中，所有`PROPAGATION_REQUIRED`修饰的**内部方法和外围方法均属于同一事务，只要一个方法回滚，整个方法都回滚**

* PROPAGATION_REQUIRED_NEW

  在外围方法**没有开启事务**的情况下，`PROPAGATION_REQUIRED_NEW`修饰的内部方法会新开启自己的事务，且**开启的事务相互独立，互补干扰**

  在外围方法**开启事务**的情况下，`PROPAGATION_REQUIRED_NEW`修饰的内部方法依然会**单独开启独立的事务**，且与外部方法的事务也独立，内部方法之间、内部方法和外部方法事务均相互独立，互补干扰

* PROPAGATION_NESTED

  在外围方法开启事务的情况下，`PROPAGATION_NESTED`修饰的内部方法属于外部事务的子事务，外围主事务回滚，子事务一定回滚，而内部子事务可以单独回滚而不影响外围主事务和其他子事务
+++
author = "杨春富"
title = "Spring中AOP简介及AspectJ五大通知注解"
date = "2021-04-06"
description = "关于Spring中AOP的简介以及基于AspectJ的五大通知注解。"
tags = [
    "Spring",
    "AOP",
    "AspectJ",
    "JAVA",
]
categories = [
    "Spring",
    "JAVA",
]
series = ["Spring"]
image = "pawel.jpg"

+++

# Spring中AOP简介及AspectJ五大通知注解

**本文以一个简单计算器为代码例子**

## 基本概念

AOP(Aspect-Oriented Programming, **面向切面编程**): 是一种新的方法论，是对传统 -OOP(Object-Oriented Programming, 面向对象编程) 的补充。

AOP 的主要编程对象是**切面**(aspect)， **切面模块化横切关注点**。

在应用 AOP 编程时, 仍然需要**定义公共功能， 但可以明确的定义这个功能用在哪里，以什么方式应用，并且不必修改受影响的类**，这样一来横切关注点就被模块化到特殊的对象(**切面**)里。

AOP 的好处：
1.  每个事物逻辑位于一个位置，代码不分散，便于维护和升级。
2.  业务模块更简洁，只包含核心业务代码。

## AOP术语
-   切面(Aspect): **横切关注点**(跨越应用程序多个模块的功能)被模块化的特殊对象。

-   通知(Advice): **切面必须要完成的工作**。

-   目标(Target): **被通知的对象**。

-   代理(Proxy): **向目标对象应用通知之后创建的对象**。

-   连接点(Joinpoint)：**程序**执行的某个特定位置：如类某个方法调用前、调用后、方法抛出异常后等。连接点由两个信息确定：方法表示的程序执行点；相对点表示的方位。方位为方法执行前的位置。

-   切点(pointcut)：每个类都拥有多个连接点：即**连接点是程序类中客观存在的事务**。**AOP** **通过切点定位到特定的连接点。类比：连接点相当于数据库中的记录，切点相当于查询条件**。切点和连接点不是一对一的关系，一个切点匹配多个连接点，切点通过 `org.springframework.aop.Pointcut` 接口进行描述，它使用类和方法作为连接点的查询条件。



## AspectJ

-   **AspectJ**：Java 社区里最完整最流行的 AOP 框架.

-   在 Spring2.0 以上版本中，可以使用基于 AspectJ 注解或基于 XML 配置的 AOP

## 应用

**本文以一个简单计算器为代码例子**

![AOP](https://gitee.com/ycfxhsw/picture/raw/master/AOP.png)


### 1、在Spring中启用AspectJ 注解支持

1.  要在 Spring 应用中使用 AspectJ 注解， **必须在** **classpath** **下包含** **AspectJ** **类库**: `aopalliance.jar、aspectj.weaver.jar 和 spring-aspects.jar`
2.  将`aopSchema`添加到 `beans`根元素中
3.  要在 Spring IOC 容器中启用 AspectJ 注解支持，只要在Bean配置文件中定义一个空的XML元素`<aop:aspectj-autoproxy>`
4.  当 Spring IOC 容器侦测到 Bean 配置文件中的 `<aop:aspectj-autoproxy>` 元素时，会自动为与 AspectJ 切面匹配的 Bean 创建代理.



### 2、用 AspectJ 注解声明切面
1、**要在Spring中声明AspectJ切面只需要在IOC容器中将切面声明为Bean实例**。当在 Spring IOC 容器中初始化 AspectJ 切面之后，Spring IOC 容器就会为那些与 AspectJ 切面相匹配的 Bean 创建代理。

2、**在AspectJ注解中**, **切面只是一个带有 `@Aspect` 注解的Java类**。

3、通知是标注有某种注解的简单的Java方法。

4、AspectJ 支持 5 种类型的通知注解:

-   **@Befores:** 前置通知，在方法执行之前执行

-   **@After:** 后置通知，在方法执行之后执行

-   **@AfterRunning:** 返回通知，在方法返回结果之后执行

-   **@AfterThrowing:** 异常通知，在方法抛出异常之后

-   **@Around:** 环绕通知，围绕着方法执行



#### 一、前置通知

-   前置通知：在方法执行之前执行的通知。

-   前置通知使用 `@Before` 注解， 并将切入点表达式的值作为注解值。

#### 二、后置通知

-   后置通知是在连接点完成之后执行的，即连接点返回结果或者抛出异常的时候，下面的后置通知记录了方法的终止。

-   一个切面可以包括一个或者多个通知。

#### 三、返回通知

-   无论连接点是正常返回还是抛出异常，后置通知都会执行。如果只想在连接点返回的时候记录日志， 应使用返回通知代替后置通知。
-   在返回通知中， 只要将 `returning` 属性添加到 `@AfterReturning` 注解中， 就可以访问连接点的返回值。 该属性的值即为用来传入返回值的参数名称。
-   必须在通知方法的签名中添加一个`同名参数`。在运行时，Spring AOP 会通过这个参数传递返回值。
-   原始的切点表达式需要出现在`pointcut`属性中。

#### 四、异常通知

-   只在连接点抛出异常时才执行异常通知。

-   将Throwing属性添加到`@AfterThrowing`注解中， 也可以访问连接点抛出的异常. Throwable 是所有错误和异常类的超类，所以在异常通知方法可以捕获到任何错误和异常。

-   如果只对某种特殊的异常类型感兴趣， 可以将参数声明为其他异常的参数类型， 然后通知就只在抛出这个类型及其子类的异常时才被执行。

#### 五、环绕通知

-   环绕通知是所有通知类型中功能最为强大的， 能够全面地控制连接点， 甚至可以控制是否执行连接点。
-   对于环绕通知来说,连接点的参数类型必须是`ProceedingJoinPoint` ，它是 `JoinPoint` 的子接口，允许控制何时执行，是否执行连接点。
-   在环绕通知中需要明确调用`ProceedingJoinPoint`的`proceed()`方法来执行被代理的方法.如果忘记这样做就会导致通知被执行了,但目标方法没有被执行。
-   注意: 环绕通知的方法需要返回目标方法执行之后的结果，即调用 `joinPoint.proceed()`的返回值，否则会出现空指针异常。

##### 实现代码
###### 1、Calc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
    <context:component-scan base-package="com.ycfxhsw.aop"></context:component-scan>

    <!-- 使用AspactJ，注解起作用：自动为匹配的类生成代理对象 -->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
</beans>
```

###### 2、Calc接口

```java
public interface Calc {
    // 加法
    int Addition(int i, int j);

    // 减法
    int Subtraction(int i, int j);

    // 乘法
    int Multiplication(int i, int j);

    // 除法
    int Division(int i, int j);
}
```

###### 3、接口实现CalcImp

```java
import org.springframework.stereotype.Component;

@Component
public class CalcImp implements Calc {
    public int Addition(int i, int j) {
        int result = i + j;
        return result;
    }

    public int Subtraction(int i, int j) {
        int result = i + j;
        return result;
    }

    public int Multiplication(int i, int j) {
        int result = i + j;
        return result;
    }

    public int Division(int i, int j) {
        int result = i + j;
        return result;
    }
}
```

###### 4、LoggingAspect

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.List;

// @Order(1) 指定切面优先级
@Component
@Aspect
public class LoggingAspect {
    /**
     * 申明切入点表达式，一般该方法内不需要添加其他方法
     * 使用 @Pointcut 申明切入点表达式
     * 后面的切入点直接使用方法名
     */
    @Pointcut("execution(public int com.ycfxhsw.aop.Calc.*(..))")
    public void DeclareJoinPointExpression() {
    }

    /**
     * 前置通知
     * @param joinPoint
     */
    @Before("execution(public int com.ycfxhsw.aop.Calc.*(int, int))")
    public void BeforeMethod(JoinPoint joinPoint) {
        String MethodName = joinPoint.getSignature().getName();
        List<Object> list = Arrays.asList(joinPoint.getArgs());
        System.out.println("Method Starts..." + MethodName + " with " + list);
    }

    /**
     * 后置通知（无论有无异常）
     * 不能访问目标方法的执行结果
     * @param joinPoint * com.ycfxhsw.aop.Calc.*(..)
     * 重用 DeclareJoinPointExpression()
     */
    @After("DeclareJoinPointExpression()")
    public void AfterMethod(JoinPoint joinPoint) {
        String MethodName = joinPoint.getSignature().getName();
        System.out.println("Method Ends..." + MethodName);
    }

    /**
     * 返回通知
     * 在方法正常结束后执行
     * 可以访问目标方法的执行结果
     * @param joinPoint,result
     */
    @AfterReturning(value = "execution(public int com.ycfxhsw.aop.Calc.*(..))", returning = "result")
    public void AfterReturningMethod(JoinPoint joinPoint, Object result) {
        String MethodName = joinPoint.getSignature().getName();
        System.out.println("Method AfterReturning..." + MethodName + " --> " + result);
    }

    /**
     * 异常通知
     * @param joinPoint,exception
     */
    @AfterThrowing(value = "execution(public int com.ycfxhsw.aop.Calc.*(..))", throwing = "exception")
    public void AfterThrowingMethod(JoinPoint joinPoint, Exception exception) {
        String MethodName = joinPoint.getSignature().getName();
        System.out.println("Method AfterThrowing..." + MethodName + " --> " + exception);
    }

    /**
     * 环绕通知，需要携带 ProceedingJoinPoint 类型的参数
     * 类似于动态代理全过程
     * 必须有返回值（目标方法返回值）
     * @param proceedingJoinPoint

     @Around("execution(public int com.ycfxhsw.aop.Calc.*(..))")
     public Object AroundMethod(ProceedingJoinPoint proceedingJoinPoint) {
     Object result = null;
     String methodName = proceedingJoinPoint.getSignature().getName();
     try {
     // 前置通知
     System.out.println("The method " + methodName + " begins with " + Arrays.asList(proceedingJoinPoint.getArgs()));
     // 执行目标方法
     result = proceedingJoinPoint.proceed();
     // 返回通知
     System.out.println("The method " + methodName + " ends with " + result);
     } catch (Throwable e) {
     // 异常通知
     System.out.println("The method " + methodName + " occurs exception:" + e);
     throw new RuntimeException(e);
     }
     // 后置通知
     System.out.println("The method " + methodName + " ends");
     return result;
     }
     */
}
```



###### 5、测试Main

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
/**
 * 测试
 */
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("Calc.xml");
        Calc calc = applicationContext.getBean(Calc.class);
        System.out.println(calc.getClass().getName());

        int resultAdd = calc.Addition(5, 6);
        System.out.println("result-->" + resultAdd + "\n");
        int resultSub = calc.Subtraction(9, 3);
        System.out.println("result-->" + resultSub + "\n");
        int resultMul = calc.Multiplication(8, 6);
        System.out.println("result-->" + resultMul + "\n");
        int resultDiv = calc.Division(99, 3);
        System.out.println("result-->" + resultDiv + "\n");
    }
}
```

### 3、指定切面的优先级

-   在同一个连接点上应用不止一个切面时，除非明确指定，否则它们的优先级是不确定的.

-   切面的优先级可以通过实现 Ordered 接口或利用 `@Order` 注解指定.

-   实现 Ordered 接口， `getOrder()` 方法的返回值越小, 优先级越高.

-   若使用 `@Order` 注解，序号出现在注解中

### 4、重用切入点定义

-   在编写 AspectJ 切面时，可以直接在通知注解中书写切入点表达式，但同一个切点表达式可能会在多个通知中重复出现。
-   在 AspectJ 切面中, 可以通过 `@Pointcut`注解将一个切入点声明成简单的方法，**切入点的方法体通常是空的**，因为将切入点定义与应用程序逻辑混在一起是不合理的。
-   切入点方法的访问控制符同时也控制着这个切入点的可见性，如果切入点要在多个切面中共用, 最好将它们集中在一个公共的类中，在这种情况下, 它们必须被声明为 `public`。在引入这个切入点时，必须将类名也包括在内，如果类没有与这个切面放在同一个包中, 还必须包含包名。
-   其他通知可以通过方法名称引入该切入点.。
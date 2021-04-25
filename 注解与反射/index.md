---
title: 注解与反射
description: 详细介绍java注解与反射的应用与原理。
date: 2021-04-23
image: zf.jpg
tags: 
    - 注解
    - 反射
    - JAVA
categories: 
    - JAVA
---

## 注解

### 注解定义

Java 注解 `Annotation` 又称 Java 标注，是 JDK5.0 引入的一种注释机制。

Java 语言中的类、方法、变量、参数和包等都可以被标注。和注释不同，Java 标注可以通过反射获取标注内容。在编译器生成类文件时，标注可以被嵌入到字节码中。Java 虚拟机可以保留标注内容，在运行 时可以获取到标注内容 。当然它也支持自定义 Java 标注。

注解与注释的区别：注解是给机器看的注释，而注释是给程序员看的提示，编译时自动忽略注释。



### 使用场景

- 编译格式检查
- 反射中解析
- 生成帮助文档
- 跟踪代码依赖
- 等等

### 内置注解

|       注解类型       |                         注解意义                         |                  补充说明                   |
| :------------------: | :------------------------------------------------------: | :-----------------------------------------: |
|      @Override       |                           重写                           |          定义在java.lang.Override           |
|     @Deprecated      |                           废弃                           |         定义在java.lang.Deprecated          |
|     @SafeVarargs     | 忽略任何使用参数为泛型变量的方法或构造函数调用产生的警告 |                java7开始支持                |
| @FunctionalInterface |                        函数式接口                        | java8开始支持，标识一个匿名函数或函数式接口 |
|     @Repeatable      |           标识某注解可以在同一个声明上使用多次           |                java8开始支持                |
|  @SuppressWarnings   |                   抑制编译时的警告信息                   |      定义在java.lang.SuppressWarnings       |

> 补充：@SuppressWarnings 的三种方法：

1. @SuppressWarnings(“unchecked”)  `抑制单类型的警告`
2. @SuppressWarnings(“unchecked”,“rawtypes”)  `抑制多类型的警告`
3. @SuppressWarnings(“all”) `抑制所有类型的警告`



## 元注解

### 定义：作用在其他注解的注解

元注解类型：

|  注解类型   | 注解意义                                                     |
| :---------: | :----------------------------------------------------------- |
| @Retention  | 标识这个注解怎么保存，是只在代码中，还是class文件中，或是在运行时可以通过反射访问 |
| @Documented | 标记这些注解是否包含在用户文档 javadoc 中                    |
|   @Target   | 标记这个注解应该是哪种 java 成员                             |
| @Inherited  | 标记这个注解是自动继承的                                     |

> Inherited 补充：
>
> - 子类会继承父类使用的注解中被 @Inherited 修饰的注解
> - 接口继承关系中，子接口不会继承父接口中的任何注解，不管父接口中使用的注解有没有被 @Inherited 修饰
> - 类实现接口时不会继承任何接口中定义的注解



## 自定义注解

#### 自定义注解框架：

![图片](https://gitee.com/ycfxhsw/picture/raw/master/an.png)

#### `Annotation` 与 `RetentionPolicy` 与 `ElementType` 。

每 1 个 Annotation 对象，都会有唯一的 RetentionPolicy 属性;至于 ElementType 属性，则有 1~n个。



#### ElementType(注解的用途类型)

“每 1 个 Annotation” 都与 “1~n 个 ElementType” 关联。当 Annotation 与某个 ElementType 关联 时，就意味着: Annotation有了某种用途。例如，若一个 Annotation 对象是 METHOD 类型，则该 Annotation 只能用来修饰方法。

|      类型       |          作用的对象类型          |
| :-------------: | :------------------------------: |
|      TYPE       |          类、接口、枚举          |
|      FIELD      |     字段属性（包括枚举常量）     |
|     METHOD      |               方法               |
|    PARAMETER    |             参数类型             |
|   CONSTRUCTOR   |             构造方法             |
| LOCAL_VARIABLE  |             局部变量             |
| ANNOTATION_TYPE |               注解               |
|     PACKAGE     |                包                |
| TYPE_PARAMETER  |          1.8之后，泛型           |
|    TYPE_USE     | 1.8之后，除了PACKAGE之外任意类型 |

```java
package java.lang.annotation;
public enum ElementType {
	TYPE,    /* 类、接口(包括注释类型)或枚举声明 */
	FIELD,     /* 字段声明(包括枚举常量) */
	METHOD,    /* 方法声明 */
	PARAMETER,   /* 参数声明 */
	CONSTRUCTOR,  /* 构造方法声明 */ 
	LOCAL_VARIABLE,  /* 局部变量声明 */ 
	ANNOTATION_TYPE,  /* 注释类型声明 */
	PACKAGE     /* 包声明 */
}
```



#### RetentionPolicy(注解作用域策略)

“每 1 个 Annotation” 都与 “1 个 RetentionPolicy” 关联。

a) 若 Annotation 的类型为 SOURCE，则意味着:Annotation 仅存在于编译器处理期间，编译器 处理完之后，该 Annotation 就没用了。例如，" @Override" 标志就是一个 Annotation。当它修 饰一个方法的时候，就意味着该方法覆盖父类的方法;并且在编译期间会进行语法检查，编译器处理完后，"@Override" 就没有任何作用了。

b) 若 Annotation 的类型为 CLASS，则意味着:编译器将 Annotation 存储于类对应的 .class 文件 中，它是 Annotation 的默认行为。

c) 若 Annotation 的类型为 RUNTIME，则意味着:编译器将 Annotation 存储于 class 文件中，并 且可由JVM读入。

| 类型    | 作用                                                         |
| :------ | :----------------------------------------------------------- |
| SOURCE  | 注解只在源码阶段保留，在编译器进行编译的时候这类注解被抹除，常见的@Override就属于这种注解 |
| CLASS   | 注解在编译期保留，但是当Java虚拟机加载class文件时会被丢弃，这个也是@Retention的**「默认值」**。@Deprecated和@NonNull就属于这样的注解 |
| RUNTIME | 注解在运行期间仍然保留，在程序中可以通过反射获取，Spring中常见的@Controller、@Service等都属于这一类 |

```java
package java.lang.annotation;
public enum RetentionPolicy {
 	SOURCE, /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该 Annotation信息了 */
 	CLASS, /* 编译器将Annotation存储于类对应的.class文件中。默认行为 */
 	RUNTIME /* 编译器将Annotation存储于class文件中，并且可由JVM读入 */
}
```



#### 定义格式

```java
@interface 自定义注解名{}
```



#### 注意事项

- 定义的注解，自动继承了java.lang,annotation.Annotation接口
- 注解中的每一个方法，实际是声明的注解配置参数
- 方法的名称就是配置参数的名称
- 方法的返回值类型，就是配置参数的类型。只能是:基本类型/Class/String/enum
- 可以通过default来声明参数的默认值
- 如果只有一个参数成员，一般参数名为value
- 注解元素必须要有值，我们定义注解元素时，经常使用空字符串、0作为默认值。



## 注解总结

运行时注解的产生作用的步骤如下：

1. 对annotation的反射调用使得动态代理创建实现该注解的一个类
2. 代理背后真正的处理对象为`AnnotationInvocationHandler`，这个类内部维护了一个map，这个map的键值对形式为`<注解中定义的方法名，对应的属性名>`
3. 任何对annotation的自定义方法的调用（抛开动态代理类继承自object的方法）,最终都会实际调用`AnnotatiInvocationHandler`的invoke方法，并且该invoke方法对于这类方法的处理很简单，拿到传递进来的方法名，然后去查map
4. map中`memeberValues`的初始化是在`AnnotationParser`中完成的，是勤快的，在方法调用前就会初始化好，缓存在map里面
5. `AnnotationParser`最终是通过`ConstantPool`对象从常量池中拿到对应的数据的，再往下`ConstantPool`对象就不深入了



## 反射

JAVA反射机制是在运行状态中，获取任意一个类的结构 , 创建对象 , 得到方法，执行方法 , 属性；

这种在运行状态动态获取信息以及动态调用对象方法的功能被称为java语言的反射机制。

### 类加载器

Java类加载器(Java Classloader)是Java运行时环境(Java Runtime Environment)的一部分，负责动态加载Java类到Java虚拟机的内存空间中。

java默认有三种类加载器，`BootstrapClassLoader、ExtensionClassLoader、App ClassLoader`。

**1、BootstrapClassLoader(引导启动类加载器):**

嵌在JVM内核中的加载器，该加载器是用C++语言写的，主要负载加载JAVA_HOME/lib下的类库，引导启动类加载器无法被应用程序直接使用。

**2、ExtensionClassLoader(扩展类加载器):**

ExtensionClassLoader是用JAVA编写，且它的父类加载器是Bootstrap。是由`sun.misc.Launcher$ExtClassLoader`实现的，主要加载JAVA_HOME/lib/ext目录中的类 库。它的父加载器是BootstrapClassLoader

**3、App ClassLoader(应用类加载器):**

App ClassLoader是应用程序类加载器，负责加载应用程序classpath目录下的所有jar和class文件。它的父加载器为Ext ClassLoader

![图片](https://gitee.com/ycfxhsw/picture/raw/master/bea.png)

类通常是按需加载，即第一次使用该类时才加载。由于有了类加载器，Java运行时系统不需要知道文件与 文件系统。学习类加载器时，掌握Java的委派概念很重要。

**双亲委派模型:**

如果一个类加载器收到了一个类加载请求，它不会自己去尝试加载这个类，而是把这个请求 转交给父类加载器去完成。每一个层次的类加载器都是如此。因此所有的类加载请求都应该传递到最顶层的 启动类加载器中，只有到父类加载器反馈自己无法完成这个加载请求(在它的搜索范围没有找到这个类) 时，子类加载器才会尝试自己去加载。委派的好处就是避免有些类被重复加载。



### 加载配置文件

1. 给项目添加resource root目录
2. 通过类加载器加载资源文件

- 默认加载的是src路径下的文件，但是当项目存在resource root目录时，就变为了加载 resource root下的文件了。
- 

### 反射获取Class

要想了解一个类,必须先要获取到该类的字节码文件对象. 在Java中，每一个字节码文件，被夹在到内存后，都存在一个对应的Class类型的对象。

**1、得到Class**

1）如果在编写代码时, 知道类的名称, 且类已经存在, 可以通过包名.类名.class 得到一个类的 类对象

2）如果拥有类的对象, 可以通过Class 对象.getClass() 得到一个类的 类对象

3）如果在编写代码时, 知道类的名称 , 可以通过Class.forName(包名+类名): 得到一个类的 类对象

上述的三种方式，在调用时。如果类在内存中不存在，则会加载到内存。如果类已经在内存中存在，不会重复加载，而是重复利用。

**2、特殊的类对象**

> 基本数据类型的类对象：基本数据类型.class
>
> 包装类.type
>
> 基本数据类型包装类对象:包装类.class



### 反射获取 Constructor

**通过class对象 获取一个类的构造方法**

1、通过指定的参数类型, 获取指定的单个构造方法 getConstructor(参数类型的class对象数组) 例如:

构造方法如下:

```java
Person(String name,int age)
```

得到这个构造方法的代码如下:

```java
Constructor c = p.getClass().getConstructor(String.class,int.class);
```

2、获取构造方法数组

```java
getConstructors();
```

3、获取所有权限的单个构造方法

```java
getDeclaredConstructor(参数类型的class对象数组)
```

4、获取所有权限的构造方法数组

```java
getDeclaredConstructors();
```

### Constructor 创建对象

```java
newInstance(Object... para)
```

调用这个构造方法, 把对应的对象创建出来。参数: 是一个Object类型可变参数, 传递的参数顺序 必须匹配构造方法中形式参数列表的顺序

```
setAccessible(boolean flag)
```

如果flag为true 则表示忽略访问权限检查 !(可以访问任何权限的方法)



### 反射获取 Method

**1、通过class对象 获取一个类的方法**

- `getMethod(String methodName , class… clss)`

根据参数列表的类型和方法名, 得到一个方法(public修饰的)

- `getMethods()`

得到一个类的所有方法 (public修饰的)

- `getDeclaredMethod(String methodName , class… clss)`

根据参数列表的类型和方法名, 得到一个方法(除继承以外所有的:包含私有, 共有, 保护, 默认)

- `getDeclaredMethods()`

得到一个类的所有方法 (除继承以外所有的:包含私有, 共有, 保护, 默认)

**2、Method 执行方法**

`invoke(Object o,Object... para) `

- 调用方法,参数1. 要调用方法的对象;参数2. 要传递的参数列表

`getName()`

- 获取方法的方法名称

`setAccessible(boolean flag)`

- 如果flag为true 则表示忽略访问权限检查 (可以访问任何权限的方法)

### 反射获取 Field

**1、通过class对象 获取一个类的属性**

- `getDeclaredField(String filedName)`

根据属性的名称, 获取一个属性对象 (所有属性)

- `getDeclaredFields()`

获取所有属性

- `getField(String filedName)`

根据属性的名称, 获取一个属性对象 (public属性)

- `getFields()`

获取所有属性 (public)

**2、Field 属性的对象类型**

常用方法:

- `get(Object o)`

参数: 要获取属性的对象 获取指定对象的此属性值

- `set(Object o , Object value)`

参数1.要设置属性值的对象；参数2.要设置的值设置指定对象的属性的值

- `getName()`

获取属性的名称

- `setAccessible(boolean flag)`

如果flag为true 则表示忽略访问权限检查 !(可以访问任何权限的属性)



### 通过反射获取注解信息

获取类/属性/方法的全部注解对象

```java
Annotation[] annotations01 = Class/Field/Method.getAnnotations();

for (Annotation annotation : annotations01) {
    System.out.println(annotation);
}
```

根据类型获取类/属性/方法的注解对象

```java
注解类型 对象名 = (注解类型) c.getAnnotation(注解类型.class);
```



### 内省 Introspector

基于反射 , java所提供的一套应用到JavaBean的API

Bean类：

- 一个定义在包中的类 ,
- 拥有无参构造器
- 所有属性私有,
- 所有属性提供get/set方法
- 实现了序列化接口

Java提供了一套java.beans包的api , 对于反射的操作, 进行了封装

![图片](https://gitee.com/ycfxhsw/picture/raw/master/jba.png)

### 1、获取Bean类信息

方法:

`BeanInfo getBeanInfo(Class cls)` 通过传入的类信息, 得到这个Bean类的封装对象 .

### 2、获取bean类的 get/set方法 数组

常用的方法:

```java
MethodDescriptor[] getPropertyDescriptors():
```

### 3、MethodDescriptor

常用方法:

- `Method getReadMethod()`

获取一个get方法

- `Method getWriteMethod();`

获取一个set方法。

有可能返回null 注意 需要加判断。
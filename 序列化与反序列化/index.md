---
title: 序列化与反序列化
description: 序列化与反序列化的简单介绍。
date: 2021-04-10
image: xulie.png
tags: 
    - 序列化
    - JAVA
categories: 
    - JAVA
---

### 一、序列化与反序列化

- 序列化：把对象转换为字节序列的过程称为对象的序列化。
- 反序列化：把字节序列恢复为对象的过程称为对象的反序列化。

**序列化的优点：**

1. 将对象转为字节流存储到硬盘上，当JVM停机的话，字节流还会在硬盘上默默等待，等待下一次JVM的启动，把序列化的对象，通过反序列化为原来的对象，并且序列化的二进制序列能够减少存储空间（永久性保存对象）。
2. 序列化成字节流形式的对象可以进行网络传输(二进制形式)，方便了网络传输。
3. 通过序列化可以在进程间传递对象。



### 二、什么时候需要用到序列化和反序列化

当我们只在本地 JVM 里运行下 Java 实例，这个时候是不需要什么序列化和反序列化的，但当我们需要将内存中的对象持久化到磁盘，数据库中时， 当我们需要与浏览器进行交互时，当我们需要实现 RPC 时， 这个时候就需要序列化和反序列化了。

前两个需要用到序列化和反序列化的场景, 是不是让我们有一个很大的疑问? 我们在与浏览器交互时，还有将内存中的对象持久化到数据库中时，好像都没有去进行序列化和反序列化, 因为我们都没有实现 Serializable 接口， 但一直正常运行。

**下面先给出结论:**

**只要我们对内存中的对象进行持久化或网络传输， 这个时候都需要序列化和反序列化.**

**理由:**

服务器与浏览器交互时真的没有用到 Serializable 接口吗？JSON 格式实际上就是将一个对象转化为字符串， 所以服务器与浏览器交互时的数据格式其实是字符串，我们来看来 String 类型的源码:

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;
```

String 类型实现了 Serializable 接口，并显示指定 serialVersionUID 的值.

然后我们再来看对象持久化到数据库中时的情况， Mybatis 数据库映射文件里的 insert 代码:

```xml
<insert id="insertUser" parameterType="com.ycfxhsw.User">
    INSERT INTO t_user(name, age) VALUES (#{name}, #{age})
</insert>
```

实际上我们并不是将整个对象持久化到数据库中， 而是将对象中的属性持久化到数据库中， 而这些属性都是实现了 Serializable 接口的基本属性.



### 三、为什么要实现 Serializable 接口？

在 Java 中实现了 Serializable 接口后， JVM 会在底层帮我们实现序列化和反序列化， 如果我们不实现 Serializable 接口， 那自己去写一套序列化和反序列化代码也行。



### 四、为什么还要指定 serialVersionUID 的值?

如果不显示指定 serialVersionUID， JVM 在序列化时会根据属性自动生成一个 serialVersionUID， 然后与属性一起序列化，再进行持久化或网络传输。

在反序列化时，JVM 会再根据属性自动生成一个新版 serialVersionUID，然后将这个新版 serialVersionUID 与序列化时生成的旧版 serialVersionUID 进行比较，如果相同则反序列化成功， 否则报错.

如果显示指定了 serialVersionUID， JVM 在序列化和反序列化时仍然都会生成一个 serialVersionUID， 但值为我们显示指定的值，这样在反序列化时新旧版本的 serialVersionUID 就一致了.

在实际开发中， 不显示指定 serialVersionUID 的情况会导致什么问题？如果我们的类写完后不再修改，那当然不会有问题。

但这在实际开发中是不可能的，我们的类会不断迭代，一旦类被修改了，那旧对象反序列化就会报错。所以在实际开发中， 我们都会显示指定一个 serialVersionUID，值是多少无所谓， 只要不变就行。

**写个实例测试下:**

**(1) User 类**

- 不显示指定 serialVersionUID.

```java
import java.io.Serializable;
public class User implements Serializable {
    String name;
    Integer age;

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

	// 省略 get/set 方法
}
```

**(2) 测试类**

- 先进行序列化， 再进行反序列化.

```java
public class SerializeTest {
    private static File file = new File("D:/Desktop/user.txt");

    private static void serialize(User user) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
        oos.writeObject(user);
        oos.close();
    }

    private static User deserialize() throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        return (User) ois.readObject();
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        User user = new User();
        user.setName("织女");
        user.setAge(18);
        System.out.println("序列化前的结果为：" + user);

        serialize(user);

        User user1 = deserialize();
        System.out.println("反序列化后的结果为：" + user1);
    }
}
```

> 序列化前的结果为：User{name='织女', age=18}
>         反序列化后的结果为：User{name='织女', age=18}

**(3) 结果**

- 先注释掉反序列化代码， 执行序列化代码， 然后 User 类新增一个属性 sex

```java
public class User implements Serializable {
    String name;
    Integer age;
    String sex;

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", sex='" + sex + '\'' +
                '}';
    }

	// // 省略 get/set 方法
}

```

- 再注释掉序列化代码执行反序列化代码， 最后结果如下:

> local class incompatible: stream classdesc serialVersionUID = -8867211101605543804, local class 	serialVersionUID = 6343365699042275217

报错结果为序列化与反序列化产生的 serialVersionUID 不一致。

接下来我们在上面 User 类的基础上显示指定一个 serialVersionUID

```java
private static final long serialVersionUID = 1L;
```

再执行上述步骤， 测试结果如下:

> 序列化前的结果为：User{name='织女', age=18} 
> 		 反序列化后的结果为：User{name='织女', age=18, sex='null'}

显示指定 serialVersionUID 后就解决了序列化与反序列化产生的 serialVersionUID 不一致的问题。



### 五、Java 序列化的其他特性

先说结论， 被 transient 关键字修饰的属性不会被序列化， static 属性也不会被序列化.

我们来测试下这个结论:

**(1) User 类**

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private Integer age;
    private transient String sex;
    private static String signature = "你眼中的世界就是你自己的样子";

    @Override
    public String toString() {
        return "User{" +
                " + name + '\\'' +
                ", age=" + age +
                ", sex='" + sex +'\\'' +
                ", signature='" + signature + '\\'' +
                '}';
    }
    
    // 省略 get/set 方法
}
```

**(2) 测试类**

```java
public class SerializeTest {
    private static File file = new File("D:/Desktop/user.txt");

    private static void serialize(User user) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
        oos.writeObject(user);
        oos.close();
    }

    private static User deserialize() throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        return (User) ois.readObject();
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        User user = new User();
        user.setName("织女");
        user.setAge(18);
        user.setSex("woman");
        System.out.println("序列化前的结果为：" + user);

        serialize(user);

        User user1 = deserialize();
        System.out.println("反序列化后的结果为：" + user1);
    }
}
```

**(3) 结果**

先注释掉反序列化代码， 执行序列化代码， 然后修改 User 类 signature = “我的眼里只有你”，再注释掉序列化代码执行反序列化代码， 最后结果如下：

> 序列化前的结果: User{name='织女', age=18, sex='woman', signature='你眼中的世界就是你自己的样子'} 
>
> 反序列化后的结果: User{name='织女', age=18, sex='null', signature='我的眼里只有你'}



### 六、static 属性为什么不会被序列化?

因为序列化是针对对象而言的，而 static 属性优先于对象存在， 随着类的加载而加载， 所以不会被序列化.

看到这个结论， 是不是有人会问， serialVersionUID 也被 static 修饰， 为什么 serialVersionUID 会被序列化？ 

其实 serialVersionUID 属性并没有被序列化， JVM 在序列化对象时会自动生成一个 serialVersionUID， 然后将我们显示指定的 serialVersionUID 属性值赋给自动生成的 serialVersionUID。


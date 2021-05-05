---
title: MyBatis4大核心对象
description: MyBatis4大核心对象SqlSessionFactoryBuiler、SqlSessionFactory、SqlSession 和 Mapper分析及生命周期。
date: 2021-05-05
tags: 
    - MyBatis
    - JAVA
categories: 
    - 数据库
    - JAVA

---

利用MyBatis来完成一次数据库操作需要经过以下步骤：

```java
/**
 * 查询所有学生信息
 */
@Test
public void queryAllStudentTest() throws IOException {
    String res = "Mybatis-Config.xml";
    // 读取 mybatis-config 配置文件
    InputStream in = Resources.getResourceAsStream(res);
    // 创建 SqlSessionFactory 对象
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
    // 创建 SqlSession 对象
    SqlSession sqlSession = sqlSessionFactory.openSession(true);
    StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
    
    List<Student> studentList = studentMapper.queryAllStudent();
    System.out.println(studentList);
    sqlSession.close();
    }
```

>1、加载配置文件
>
>2、获取 SqlSessionFactoryBuiler 对象
>
>3、通过 SqlSessionFactoryBuiler 和配置文件流来获取 SqlSessionFactory 对象
>
>4、利用 SqlSessionFactory 对象来打开一个 SqlSession
>
>5、通过 SqlSession 来获得对应的Mapper对象
>
>6、通过 Mapper 对象调用对应接口来查询数据库
>
>完成后关闭 SqlSession

从这些步骤我们可以看到，MyBatis完成一次数据库操作主要有4大核心对象：`SqlSessionFactoryBuiler，SqlSessionFactory，SqlSession 和 Mapper。`



### 一、SqlSessionFactoryBuilder（构造器）

- SqlSessionFactoryBuilder 的唯一作用就是用来创建 SqlSessionFactory，创建完成之后就不会用到它了，所以SqlSessionFactoryBuiler生命周期极短。


- SqlSessionFactoryBuiler中提供了9个方法，返回的都是SqlSessionFactory对象。


- SqlSessionFactoryBuiler会根据配置或者代码来生成 SqlSessionFactory，采用的是分步构建的 Builder 模式。


#### (2+1)方法分析

虽然说提供了9个方法，但是实际上我们可以看成(2+1)个方法。前面三个方法最终都调用了第4个方法，所以实际上就是一个方法:

```java
public SqlSessionFactory build(Reader reader) {
    return build(reader, null, null);
}

public SqlSessionFactory build(Reader reader, String environment) {
    return build(reader, environment, null);
}

public SqlSessionFactory build(Reader reader, Properties properties) {
    return build(reader, null, properties);
}

public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());   
      ......
  }
```

在看下面这个其实也是一回事，有三个方法最终也是调用了同一个方法：

```java
public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
}

public SqlSessionFactory build(InputStream inputStream, String environment) {
    return build(inputStream, environment, null);
}

public SqlSessionFactory build(InputStream inputStream, Properties properties) {
    return build(inputStream, null, properties);
}

public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());
	......
}
```

所以这两个方法就对应了2+1方法中的2。下面这个就是2+1中的1：

```java
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

在上面可以看到，这2个方法在将配置文件解析成一个Java配置类Configuration之后，又同时调用了另一个方法build(Configuration config),而这个方法也没做什么事,就是把配置类传进去返回了一个DefaultSqlSessionFactory对象，这个对象就是SqlSessionFactory的实现类.

所以说2+1中的1没什么好分析的，那么这个2其实就是支持了2种读取文件的方式，比如说我们获取一个SqlSessionFactory对象最常用的是通过如下方式：

```java
String res = "Mybatis-Config.xml";
// 读取 mybatis-config 配置文件
InputStream in = Resources.getResourceAsStream(res);
// 创建 SqlSessionFactory 对象
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
```

但是其实我们同样可以采用下面的方式才获取SqlSessionFactory对象：

```java
Reader reader = Resources.getResourceAsReader(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
```

这两个方法的唯一区别就是在这里，另外两个参数都是一样的。

- String environment：

    这个其实就是用来决定加载哪个环境的。
    下面就是一个mybatis-config文件，有定义了一个environment id，当我们需要支持多环数据源切换的时候可以定义多个id，然后根据不同环境传入不同id即可。

    ```xml
    <!-- 配置数据库信息 -->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property value="${jdbc_driver}" name="driver"/>
                <property value="${jdbc_url}" name="url"/>
                <property value="${jdbc_username}" name="username"/>
                <property value="${jdbc_password}" name="password"/>
            </dataSource>
        </environment>
    </environments>
    ```

- Properties properties：

- 上面dataSource内的属性没有直接输入，而是采用了${jdbc.driver}来配置，而这些属性的值我们可以单独维护在一个properties文件内。

```properties
jdbc_driver=com.mysql.cj.jdbc.Driver
jdbc_url=jdbc:mysql://localhost:3306/school?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC
jdbc_username=root
jdbc_password=xxxxxx
```

然后代码中就可以这么改写：

```java
Reader reader = Resources.getResourceAsReader(resource);
Properties properties = Resources.getResourceAsProperties("db.properties");
SqlSessionFactory sSFactory = new SqlSessionFactoryBuilder().build(reader,properties);
```

采用这种方式之后，mybatis-config.xml文件可以直接用${}来取值了。当然其实还有一种更简单的方式，那就是直接在mybatis-config.xml引入properties文件，这样就不用在代码中去解析properties文件了：

```properties
<properties resource="db.properties"></properties>
```

build()方法的功能就是去读取XML文件，并将读取到的信息封装到Configuration中。

通过上面的分析我们也很容易的知道，既然提供了一个方法可以直接传入Configuration来获取SqlSessionFactory对象，那么其实我们也可以不采用XML形式，转而直接通过Java代码的方式来直接构建Configuration,然后直接获取SqlSessionFactory对象。如下：

```java
Configuration congiguration = new Configuration();
congiguration.setDatabaseId("xxx");
congiguration.setCacheEnabled(false);
SqlSessionFactory sqlSessionFactory1 = new SqlSessionFactoryBuilder().build(congiguration);
```

当然，这种方式如果要修改的话每次都要改代码，会比较麻烦，而且看起来比较不直观，所以还是推荐使用xml方式来进行MyBatis的基础属性配置。

#### InputStream和Reader

Reader和InputStream都是Java中I/O库提供的两套读取文件方式。Reader用于读入16位字符，也就是Unicode 编码的字符；而 InputStream 用于读入ASCII字符和二进制数据。



### 二、SqlSessionFactory（工厂接口）

- SqlSessionFactory是一个接口，默认的实现类，主要是用来生成SqlSession对象，而SqlSession对象是需要不断被创建的，所以SqlSessionFactory是全局都存在的，也没有必要重复创建，所以这是一个单例对象。


- SqlSessionFactory看名字就可以很容易想到，用到的是工厂设计模式。


- SqlSessionFactory只有两个实现类：DefaultSqlSessionFactory和SqlSessionManager。




#### DefaultSqlSessionFactory

DefaultSqlSessionFactory 类提供了8个方法用来获取SqlSession对象。方法列举如下：

```java
SqlSession openSession();

SqlSession openSession(boolean autoCommit);
SqlSession openSession(Connection connection);
SqlSession openSession(TransactionIsolationLevel level);

SqlSession openSession(ExecutorType execType);
SqlSession openSession(ExecutorType execType, boolean autoCommit);
SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
SqlSession openSession(ExecutorType execType, Connection connection);
```

这8个方法中TransactionIsolationLevel和autoCommit都比较好理解，就是事务隔离级别和是否开启自动提交。
前面6个方法最终实际上调用了openSessionFromDataSource方法，而后面2个又是调用了openSessionFromConnection方法。

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        // 获取配置好的环境
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        final Executor executor = configuration.newExecutor(tx, execType);
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        // may have fetched a connection so lets call close()
        closeTransaction(tx); 
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

```java
private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
        boolean autoCommit;
        try {
            // 获取事务是否开启自动提交
            autoCommit = connection.getAutoCommit();
        } catch (SQLException e) {
          // Failover to true, as most poor drivers
          // or databases won't support transactions
          autoCommit = true;
      }
      // 获取配置环境
      final Environment environment = configuration.getEnvironment();
      // 获取事务对象
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      final Transaction tx = transactionFactory.newTransaction(connection);
      // 获取执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 构造一个SqlSession对象返回
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```


从上面的两个方法中很明显可以看到，这两个方法基本一样，唯一的区别是openSessionFromConnection方法是从Connection对象中获取当前会话是否需要开启自动提交。

#### Executor

MyBatis中最终去执行sql语句是通过Executor对象实现的，Executor只有三种类型：`SIMPLE, REUSE, BATCH。`

- 默认使用的是SIMPLE类型对应的SimpleExecutor执行器。SimpleExecutor表示普通的执行器，不会做任何处理。

- REUSE类型对应的是ReuseExecutor执行器，会复用预处理语句。

- BATCH类型对应的是BatchExecutor执行器，一般用来执行批量操作。

Executor是属于SqlSession的四大对象之一，后面会有一篇博客来专门分析，在这里我们暂时不深入展开。



#### SqlSessionManager

SqlSessionManager其实也不是什么新东西，只是同时把SqlSessionFactory和SqlSession的职责放到了一起，所以这个我们不过多的讨论。

```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {
    private final SqlSessionFactory sqlSessionFactory;
    private final SqlSession sqlSessionProxy;
    .....
}
```



### 三、SqlSession（会话）

一个既可以发送 SQL 执行返回结果，也可以获取 Mapper 的接口。在现有的技术中，一般我们会让其在业务逻辑代码中“消失”，而使用的是 MyBatis 提供的 SQL Mapper 接口编程技术，它能提高代码的可读性和可维护性。

SqlSession是用来操作xml文件中我们写好的sql语句，每次操作数据库我们都需要一个SqlSession对象，SqlSession是用来和数据库中的事务进行对接的，所以SqlSession里面是包含了事务隔离级别等信息的。

SqlSession实例是线程不安全的，故最佳的请求范围是请求(request)或者方法(method)。

SqlSession也是一个接口，有两个实现类：DefaultSqlSession和SqlSessionManager。SqlSessionManager上面有提到，这里就不重复了。



### 四、Mapper（映射器）

Mapper是一个接口，没有任何实现类。主要作用就是用来映射Sql语句的接口，映射器的接口实例从SqlSession对象中获取，所以说Mapper实例作用域是和SqlSession相同或者更小。Mapper实例的作用范围最好是保持在方法范围，否则会难以管理。

Mapper接口的名称要和对应sql语句的xml文件同名，Mapper接口中定义的方法名称对应了xml文件中的语句id。

#### 四大对象生命周期

SqlSessionFactoryBuiler只需要在创建SqlSessionFactory对象的时候使用，创建完成之后即可被丢弃。SqlSessionFactory全局唯一，是一个单例对象，但是需要全局存在。SqlSession一般对应了一个request，Mapper一般控制在方法内。

| 对象                    | 生命周期                                          |
| ----------------------- | ------------------------------------------------- |
| SqlSessionFactoryBuiler | 方法局部（Method）使用完成即可被丢弃              |
| SqlSessionFactory       | 应用级别（Application），全局存在，是一个单例对象 |
| SqlSession              | 请求或方法（Request / Method）                    |
| Mapper                  | 方法（Method）                                    |
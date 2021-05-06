---
title: MyBatis中SqlSession下四大对象
description: MyBatis中SqlSession下四大对象Executor、StatementHandler、ParameterHandler、ResultSetHandler的分析及理解。
date: 2021-05-06
tags: 
    - MyBatis
    - JAVA
categories: 
    - 数据库
    - JAVA
---

我们在执行Sql之前，需要先获取SqlSession对象，但是SqlSession下面还有四大对象，所以SqlSession只是个甩手掌柜，真正干活的却是Executor等四大对象：`Executor、StatementHandler、ParameterHandler、ResultSetHandler`。

## MyBatis架构分层

首先我们先来建立一个MyBatis的整体认知，下面就是MyBatis的一个整体分层架构图：

![](https://gitee.com/ycfxhsw/picture/raw/master/MyBatisjiagou.png)

#### 接口层
接口层的核心对象就是SqlSession，SqlSession是应用和MyBatis打交道的桥梁，SqlSession上定义了一系列数据库操作方法，然后在收到请求的时候再去调用核心处理层模块来完成具体操作。
#### 核心处理层
真正和数据库相关操作都是在核心层完成的，核心层主要做了以下4件事：

1. 将接口中传入的参数解析并且映射成为JDBC。
2. 解析xml文件中的SQL语句，包括参数的插入和动态SQL的生成。
3. 执行SQL语句。
4. 处理结果集，并且映射成Java对象。

**PS：**插件也属于核心层，因为插件就是拦截核心处理层对象。

#### 基础支持层
基础支持层就是封装一些底层操作用来处理核心层的功能。

我们今天要讲解的四大对象就是核心处理层的四大对象，接下来就让我们逐一进行分析。

## 一、Executor

Executor就是真正用来执行Sql语句的对象，我们调用SqlSession中的方法，最终实际上都是通过Executor来完成的。我们先来看一下Executor的类图关系：

![](https://gitee.com/ycfxhsw/picture/raw/master/executor.png)

这里面其实用到了模板方法模式。顶层接口Executor定义了一系列规范，而在抽象类BaseExecutor中将一些固定不变的方法进行了封装，并定义了一下抽象方法待子类实现。

### BaseExecutor

BaseExecutor是一个抽象类，除了下面的四个方法是抽象方法，其余所有方法都是一些如获取缓存，事务提交，获取事务等公共操作，所以就直接被实现了。
如下代码所示，这四个方法就是抽象方法：

```java
protected abstract int doUpdate(MappedStatement ms, Object parameter) throws SQLException;

protected abstract List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException;

protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException;

protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException;
```

- doFlushStatements()：刷新Statement对象

- doQuery()：执行查询语句并返回List

- doQueryCursor()：执行查询语句并返回Cursor对象

- doUpdate()：执行更新操作

MyBatis核心配置文件中的setting标签内有一个属性`defaultExecutorType`，有三种执行类型：`SIMPLE，REUSE，BATCH`。如果不配置则默认就是SIMPLE。这三种类型就是对应了BaseExecutor的三个子类：
**SimpleExecutor**，**ReuseExecutor**和**BatchExecutor**。

#### SimpleExecutor

SimpleExecutor是最简单的一个执行器，没有任何特殊的，就是实现了BaseExecutor中的四个抽象方法。
我们来看其中一个doQuery()方法，可以看到没有任何特殊逻辑，就是很常规的流程操作：

```java
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        // 获取配置文件对象
        Configuration configuration = ms.getConfiguration();
        // 获取 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // 初始化 Statement 对象，并进行参数设置
        stmt = prepareStatement(handler, ms.getStatementLog());
        // 调用 StatementHandler 执行查询
        return handler.query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}
```

其中初始化Statement对象我们为了对比，也进去看一下：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取连接
    Connection connection = getConnection(statementLog);
    // 处置化 Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 参数化，参数设置
    handler.parameterize(stmt);
    return stmt;
}
```

我们再来看一个doFlushStatements()方法:

```java
@Override
public List<BatchResult> doFlushStatements(boolean isRollback) {
    return Collections.emptyList();
}
```

这里什么都没做，直接返回了一个空List。

#### ReuseExecutor
ReuseExecutor相比较于SimpleExecutor做了一点优化，那就是将Statement对象进行了缓存处理，不会每次都创建Statement对象，这样做的话减少了SQL预编译和创建对象的开销。

ReuseExecutor中的查询和更新方法和SimpleExecutor完全一样，而其中的差别就在于创建Statement对象上，我们进去ReuseExecutor的prepareStatement方法：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();
    if (hasStatementFor(sql)) {
        // 从缓存中获取
        stmt = getStatement(sql);
        applyTransactionTimeout(stmt);
    } else {
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection, transaction.getTimeout());
        putStatement(sql, stmt);
    }
    // 参数设置
    handler.parameterize(stmt);
    return stmt;
}
```

我们可以看到区别就是多了一个从缓存中获取Statement对象的逻辑，用来达到复用Statement对象的目的。其中getStatement是通过ReuseExecutor内的一个HashMap属性来获取Statement对象，其中key值就是我们执行的sql语句：

```java
private final Map<String, Statement> statementMap = new HashMap<>();
```

我们再来看看doFlushStatements方法，可以看到，这里面会遍历map将Statement关闭，并清空map，看到这里，大家应该就明白了为什么SimpleExecutor内这个方法直接返回的是空，因为SimpleExecutor方法没有Statement需要关闭。

```java
@Override
public List<BatchResult> doFlushStatements(boolean isRollback) {
    for (Statement stmt : statementMap.values()) {
        closeStatement(stmt);
    }
    statementMap.clear();
    return Collections.emptyList();
}
```

**PS：**doFlushStatements方法在BaseExecutor中的commit(),rollback(),close()方法中会被调用(即：事务提交，事务回滚，事务关闭三个方法)。

#### BatchExecutor
BatchExecutor从名字上也可以看出来，这是一个支持批量操作的执行器。

如果说大家都用过jdbc就知道，jdbc是支持批量操作的，有一个executeBatch()方法用来执行批量操作，但是有一个前提就是执行批量操作的sql除了参数不同，其他都应该是相同的（关于这一点，下面我们会举例来说明）。
需要注意的是，**批量操作只支持insert,update,delete语句，select语句是不支持的**，所以BatchExecutor内的doQuery方法和其他执行器并没有很大不同，区别就是在查询之前会先调用flushStatements()，我们不做过多讨论，主要看一下doUpdate方法：

```java
@Override
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // 如果当前执行的SQL与上次执行的SQL相同，且 MappedStatement对象也相同
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
        int last = statementList.size() - 1;
        stmt = statementList.get(last);
        applyTransactionTimeout(stmt);
        handler.parameterize(stmt);//fix Issues 322
        // 获取缓存中最后一次的 batchResult
        BatchResult batchResult = batchResultList.get(last);
        // 添加参数
        batchResult.addParameterObject(parameterObject);
    } else {
        Connection connection = getConnection(ms.getStatementLog());
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);    //fix Issues 322
        currentSql = sql;
        currentStatement = ms;
        statementList.add(stmt);
        batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    // 调用JDBC的 PreparedStatement 中 addBatch()方法
    handler.batch(stmt);
    // 因为这个方法不知道成功了多少条，返回一个负数就行
    // BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002
    return BATCH_UPDATE_RETURN_VALUE;
}
```

下面是一些成员属性：

```java
// 缓存执行的Statement，一个Statement内可能会有多个相同SQL  
private final List<Statement> statementList = new ArrayList<>();
// 记录批处理的结果
private final List<BatchResult> batchResultList = new ArrayList<>();
// 当前执行SQL
private String currentSql;
// 当前执行的MappedStatement
private MappedStatement currentStatement;
```

这个方法的逻辑就是判断相同模式的sql会共用同一个Statement对象，然后缓存到list内，需要注意的是它只会和前一个进行比对，也就是说假如你有相同模式的2条sql，但是你中间先执行了一条其他sql，那么就会产生3个Statement对象，从而无法共用了。

**PS：**上面的doUpdate中返回了一个数：BATCH_UPDATE_RETURN_VALUE，这个数其实没有什么特别含义，只需要返回一个没有意义的负数就可以，表示代码不知道执行成功多少条。比如说直接返回-1，或者干脆直接返回Integer.MIN_VALUE都是没有问题的，全凭个人喜好了。

接下来我们再看看doFlushStatements()方法：

```java
@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
        List<BatchResult> results = new ArrayList<>();
        if (isRollback) {
            return Collections.emptyList();
        }
        // 遍历之前存储好的 Statement
        for (int i = 0, n = statementList.size(); i < n; i++) {
            Statement stmt = statementList.get(i);
        	applyTransactionTimeout(stmt);
        	BatchResult batchResult = batchResultList.get(i);
        	try {
                // 调用JDBC的executeBatch方法执行批量更新，并设置更新条数
                batchResult.setUpdateCounts(stmt.executeBatch());
                MappedStatement ms = batchResult.getMappedStatement();
          	    List<Object> parameterObjects = batchResult.getParameterObjects();
                KeyGenerator keyGenerator = ms.getKeyGenerator();
            if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
            	jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
            } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
              for (Object parameter : parameterObjects) {
                  keyGenerator.processAfter(this, ms, stmt, parameter);
              }
          }
    ......
    ......
  }
```

这个方法就是去遍历上面存储好的Statement，依次调用Statement中的executeBatch方法。

##### 三种常用批量插入方式

讲到这里，顺便聊一聊MyBatis编程中常用的三种批量操作方式。

###### 1️⃣直接代码循环

这是最简单的一种，但也是效率最低的一种，如下简单示例：

```java
UserAddressMapper userAddressMapper = session.getMapper(UserAddressMapper.class);
for (UserAddress userAddress : userAddressList){
     userAddressMapper.insert(userAddress);
 }
```

这种方式会把大部分时间消耗在网络连接通信上，一般不建议使用。

###### 2️⃣利用MyBatis中批量标签foreach处理

新建测试类：

```java
public class TestBatchInsert {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        //读取mybatis-config配置文件
        InputStream inputStream = Resources.getResourceAsStream(resource);
        //创建SqlSessionFactory对象
        SqlSessionFactory sSFactory = new SqlSessionFactoryBuilder().build(inputStream);
        //创建SqlSession对象
        SqlSession session = sSFactory.openSession();
        try {
            List<UserAddress> userAddressList = new ArrayList<>();
            
            UserAddress userAddr = new UserAddress();
            userAddr.setAddress("广东深圳");
            userAddressList.add(userAddr);
            
            UserAddress userAddr2 = new UserAddress();
            userAddr2.setAddress("广东广州");
            userAddressList.add(userAddr2);

            UserAddressMapper userAddressMapper = session.getMapper(UserAddressMapper.class);

            userAddressMapper.batchInsert(userAddressList);
            session.commit();
        }finally {
            session.close();
        }
    }
}
```

Mapper接口新增如下方法：

```java
int batchInsert(List<UserAddress> userAddresses);
```

```xml
 <insert id="batchInsert">
        insert into lw_user_address (address) values
       <foreach collection="list" item="item" separator=",">
           (#{item.address})
       </foreach>
    </insert>
```

`foreach`顺便我们介绍一下**foreach标签**的用法：

**collection：**表示待循环的对象。当参数为List时，默认"list"，参数为数组时，默认"array"。但是当我们在Mapper接口中使用@Param(“xxx”)时，默认的list,array将会失效，必须使用我们自己设置的参数名。 还有一种特殊情况就是假如集合里面有集合或者对象里面有集合，那么可以使用collection=“xxx.属性名”。

**item：**表示当前循环中的元素。

**open/close：**表示循环体开始和结束位置插入的符号，一般成对出现，in语句使用较多,如：

```xml
<select id="test">
       select * from xxx where id in 
     <foreach collection="list" item="item" open="(" close=")" separator=",">
         #{item.xxx}
     </foreach>
   </select>
```

**separator：**表示每个循环之后的分割符号，可参考上面的例子

**index：**当前元素在集合的下标，如果是map则是map的key值，这个参数一般用的相对较少。

###### 3️⃣BatchExecutor插入

我们把上面的普通例子中获取Session的例子改写一下：

```java
SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);
```

可以看到，这两条语句就是相同模式的sql，只是参数不同，所以直接执行一次。

我们把上面的例子改写一下：

```java
UserAddress userAddr = new UserAddress();
userAddr.setAddress("广东深圳");
userAddr.setId(1);
userAddressList.add(userAddr);

UserAddress userAddr2 = new UserAddress();
userAddr2.setAddress("广东广州");
userAddr2.setId(2);
userAddressList.add(userAddr2);

UserAddressMapper userAddressMapper = session.getMapper(UserAddressMapper.class);

userAddressMapper.insert(userAddr);//sql-1
userAddressMapper.insert10(userAddr2);//sql-10
userAddressMapper.insert(userAddr);//sql-1
```


insert和insert10分别对应如下语句（一条是1个参数，一条是2个参数）：

```xml
<insert id="insert" parameterType="com.mybatis.model.UserAddress" useGeneratedKeys="true" keyProperty="address">
    insert into lw_user_address (address) values (#{address})
</insert>

<insert id="insert10" parameterType="com.mybatis.model.UserAddress" useGeneratedKeys="true" keyProperty="address">
    insert into lw_user_address (id,address) values (#{id},#{address})
</insert>
```

上面就是有两种sql模型，理论上应该执行2次，但是我们根据源码知道，因为insert语句中间被insert10隔开了，所以实际上sql-1也是不能复用的，也就是会执行3次：

**PS:**这三种批量执行的效率有兴趣的可以自己去测试一下，效率最高的应该是foreach标签的形式，网上有其他

#### ClosedExecutor

ClosedExecutor是ResultLoaderMap(懒加载时会使用)内的一个内部类，没有任何具体实现，一般我们不会主动去使用。

### CachingExecutor

这个执行器和缓存有关，在这里我们先不展开，后面讲述缓存实现原理的时候再来分析。

## 二、StatementHandler

StatementHandler是数据库会话器，专门用来处理数据库会话的。StatementHandler内运用了适配器模式和策略模式的思想。
类图结构和Executor非常相似，如下图所示：

![](https://gitee.com/ycfxhsw/picture/raw/master/statement.png)

这个接口中的方法也相对较少，prepare方法是用来初始化具体Statement对象的：

```java
public interface StatementHandler {
  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;

  void parameterize(Statement statement)
      throws SQLException;

  void batch(Statement statement)
      throws SQLException;

  int update(Statement statement)
      throws SQLException;

  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;

  BoundSql getBoundSql();

  ParameterHandler getParameterHandler();
}
```



### BaseStatementHandler

BaseStatementHandler是一个抽象类，实现了StatementHandler中的所有方法，只留下了一个初始化Statement对象方法留给子类实现。

#### SimpleStatementHandler

SimpleStatementHandler对应JDBC的Statement，是一种非预编译语句，所以参数中是没有占位符的，相当于参数中会用$符号。

#### PreparedStatementHandler

PreparedStatementHandler对应JDBC的PrepareStatement语句，是一种预编译，参数会有占位符，预编译可以防止SQL注入。

#### CallableStatementHandler

CallableStatementHandler依赖于JDBC的Callablement，用来调用存储过程语句。

### RoutingStatementHandler

RoutingStatementHandler这个从名字上可以看出来，只是起到了一个路由作用，会根据statement类型来生成相对应的Statement对象：

```java
private final StatementHandler delegate;

public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    switch (ms.getStatementType()) {
        case STATEMENT:
            delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        	break;
      	case PREPARED:
        	delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        	break;
      	case CALLABLE:
        	delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        	break;
      	default:
        	throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
}
......
```



##  三、ParameterHandler

ParameterHandler是一个参数处理器，主要是用来对预编译语句进行参数设置额，只有一个默认实现类DefaultParameterHandler。ParameterHandler中只定义了两个方法，一个获取参数，一个设置参数：

```java
public interface ParameterHandler {
  Object getParameterObject();

  void setParameters(PreparedStatement ps)
      throws SQLException;
}
```



## 四、ResultSetHandler

ResultHandler是一个结果处理器，StatementHandler完成了查询之后，最终就是通过ResultHandler来实现结果集映射，ResultSetHandler接口中只定义了3个方法用来处理结果，而这三个方法对应了三种返回结果：

```java
public interface ResultSetHandler {
	// 处理结果集，返回相应的结果集对象
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
	// 处理结果集，返回 Cursor 对象
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;
	// 处理存储过程的游标参数
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

ResultHandler也默认提供了一个实现类：DefaultResultSetHandler。一般我们平常用的最多的就是通过handleResultSets来实现结果集转换，这个方法的大致思路我们上一篇文章已经分析过了，在这里就不重复展开。

## 总结

经过这篇文章的分析，我想大家可以体会到SqlSession只是个甩手掌柜的意思，因为SqlSession只是一个对外接口，实际干活的却是Executor等四大对象：**Executor,StatementHandler,ParameterHandler,ResultSetHandler。**本文的重点讲述了Executor对象，并对比了三种常用批量操作的使用方法，相信通过这篇文章的学习大家对MyBatis的执行流程可以有更深一步的了解，掌握了这四大对象，后面就会更容易理解MyBatis的插件实现原理。


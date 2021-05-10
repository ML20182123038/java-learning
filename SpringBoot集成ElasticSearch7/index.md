---
title: SpringBoot集成ElasticSearch 7
description: SpringBoot集成ElasticSearch 7，操作介绍。
date: 2021-05-10
image: wallroom7.jpg
tags: 
    - ElasticSearch 
    - SpringBoot
    - JAVA
categories: 
    - Spring
    - SpringBoot
    - JAVA
    - 开发者
---

本文基于[Java High Level REST Client 7.12](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html)

## 一、引入依赖

```xml
<!-- elasticsearch版本，保持与本地版本一致 -->
<elasticsearch.version>7.12.1</elasticsearch.version>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```



## 二、配置 RestHighLevelClient

```java
/**
 * 配置 RestHighLevelClient
 */
@Configuration
public class ElasticSearchConfig {
    @Bean
    public RestHighLevelClient restHighLevelClient() {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(
                        new HttpHost("localhost", 9200, "http")));
        return client;
    }
}
```

**可以去配置文件中直接设置**

## 三、实例

`本文没有使用 ElasticSearchTemplate、ElasticSearchRestTemplate`

### 索引操作

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

/**
* 创建索引
*/
@Test
public void createIndex() throws IOException {
    // 创建索引请求
    CreateIndexRequest indexRequest = new CreateIndexRequest("test4");
    // 客户端执行请求，请求后获得响应
    CreateIndexResponse indexResponse = restHighLevelClient.indices().create(indexRequest, RequestOptions.DEFAULT);
        System.out.println(indexResponse.index()); //输出：test4
    }

/**
 * 获取索引，只能判断索引存不存在
 */
@Test
public void existIndex() throws IOException {
    // 创建获取索引请求
    GetIndexRequest request = new GetIndexRequest("test4");
    boolean exists = restHighLevelClient.indices().exists(request, RequestOptions.DEFAULT);
    System.out.println(exists);
}

/**
 * 删除索引
 */
@Test
public void deleteIndexTest() throws IOException {
    DeleteIndexRequest request = new DeleteIndexRequest("test4");
    AcknowledgedResponse delete = restHighLevelClient.indices().delete(request, RequestOptions.DEFAULT);
    System.out.println(delete.isAcknowledged());
}
```

### 文档操作

**1、先定义一个实体类User：**

```java
public class User {
    private String name;
    private Integer age;
    
    // 其他省略
}
```

**2、引入fastjson依赖**

```xml
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.75</version>
</dependency>
```

**3、文档基本操作**

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

/**
 * 添加文档操作
 */
@Test
public void addDocumentTest() throws IOException {
    User user = new User("织女", 18);
    // 拿到索引
    IndexRequest request = new IndexRequest("test4");
    // 设置文档id
    request.id("1");
    // 将User对象转化为JSON，数据放入请求
    request.source(JSON.toJSONString(user), XContentType.JSON);
    // 客户端发送请求后获取响应
    IndexResponse index = restHighLevelClient.index(request, RequestOptions.DEFAULT);

    System.out.println(index.toString());
    // 索引状态
    System.out.println(index.status());
}

/**
 * 获取文档，判断是否存在
 */
@Test
public void existDocument() throws IOException {
    GetRequest request = new GetRequest("test4","1");
    boolean exists = restHighLevelClient.exists(request,RequestOptions.DEFAULT);
    System.out.println(exists);
}

/**
 * 获取文档信息
 */
@Test
public void getDocInfo() throws IOException {
    GetRequest request = new GetRequest("test4", "1");
    GetResponse response = restHighLevelClient.get(request, RequestOptions.DEFAULT);
    // 从响应中获取各种信息
    System.out.println(response.getSourceAsString());
    System.out.println(response.getVersion());
    System.out.println(response);
}

/**
 * 更新文档信息
 */
@Test
public void updateDocument() throws IOException {
    UpdateRequest request = new UpdateRequest("test4", "1");
    
    User user = new User("老牛", 20);
    request.doc(JSON.toJSONString(user), XContentType.JSON);
    
    UpdateResponse response = restHighLevelClient.update(request, RequestOptions.DEFAULT);
    System.out.println(response.status());
    System.out.println(response.toString());
}

/**
 * 删除文档
 */
 @Test
 public void deleteDocument() throws IOException {
    DeleteRequest request = new DeleteRequest("test4", "2");
    DeleteResponse response = restHighLevelClient.delete(request, RequestOptions.DEFAULT);
    System.out.println(response.status());
}
```

**4、批量插入、更新、删除**

```java
/**
 * 批量插入
 */
@Test
public void bulkTest() throws IOException {
    BulkRequest request = new BulkRequest();

    List<User> list = new ArrayList<>();
    list.add(new User("大一", 18));
    list.add(new User("大二", 19));
    list.add(new User("大三", 17));
    list.add(new User("大四", 11));
    list.add(new User("大五", 12));
    list.add(new User("大六", 13));
    list.add(new User("大七", 14));
    list.add(new User("大八", 15));
    list.add(new User("大九", 16));

    // 不设置id就会随机生成
    // 批处理请求
    for (int i = 0; i < list.size(); i++) {
        // 批量更新、删除也在这里操作
        request.add(new IndexRequest("test4")
                .id("" + i)
                .source(JSON.toJSONString(list.get(i)), XContentType.JSON));
    }
    BulkResponse bulk = restHighLevelClient.bulk(request, RequestOptions.DEFAULT);
    System.out.println(bulk.status());
}
```

**5、搜索查询**

```java
/**
 * 搜索查询
 */
@Test
public void searchTest() throws IOException {
    SearchRequest request = new SearchRequest();

    // 条件构造
    SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
    // 查询条件，用 QueryBuilders 工具类
    // termQuery 精确查询
    // 有中文需要在字段后加上 .keyword，如下 name.keyword
    TermQueryBuilder termQuery = QueryBuilders.termQuery("name.keyword", "大二");
    // 或者使用 matchQuery 精确匹配
    // MatchQueryBuilder matchQuery = QueryBuilders.matchQuery("name", "大三");
    
    sourceBuilder.query(termQuery);
    // 高亮、分页等等也在这写
    // 高亮构造，但是后面还需解析高亮
    // HighlightBuilder highlightBuilder = new HighlightBuilder();
    // sourceBuilder.highlighter(highlightBuilder);

    // 把资源放入 request
    request.source(sourceBuilder);
    
    SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
    SearchHits hits = response.getHits();
    System.out.println(JSON.toJSONString(hits));
}
```



### 实体注解

```java
@Document(indexName = "goods",type = "detail",shards = 3,replicas = 1)
public class EsGoods {
    @Id
    Long id;
    @Field(type = FieldType.Text,analyzer = "ik_max_word")
    String title; //标题
    @Field(type = FieldType.Keyword)
    String category;// 分类
    @Field(type = FieldType.Keyword)
    String brand; // 品牌
    @Field(type = FieldType.Double)
    Double price; // 价格
    @Field(index = false, type = FieldType.Keyword)
    String images; // 图片地址
}



@Document 作用在类，标记实体类为文档对象，一般有四个属性

　　indexName：对应索引库名称

　　type：对应在索引库中的类型

　　shards：分片数量，默认5

　　replicas：副本数量，默认1

@Id 作用在成员变量，标记一个字段作为id主键

@Field 作用在成员变量，标记为文档的字段，并指定字段映射属性：

　　type：字段类型，取值是枚举：FieldType

　　index：是否索引，布尔类型，默认是true

　　store：是否存储，布尔类型，默认是false

　　analyzer：分词器名称：ik_max_word

IK分词器有两种分词模式：ik_max_word和ik_smart模式。
 ik_max_word： 会将文本做最细粒度的拆分
 ik_smart：会做最粗粒度的拆分
```




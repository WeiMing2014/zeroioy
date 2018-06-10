---
layout: splash
title:  "Java 利用Rest Api 请求 ElasticSearch"
date:   2018-06-10 00:00:00 +0800
categories: jekyll elasticsearch
excerpt : "调研 Java 利用用Rest Api 请求 ElasticSearch 的方法"
---

近期在调研全文搜索的选型，接触了ElasticSearch,接触了四种Java请求ElasticSearch的方式:
1. Spring boot data elasticsearch
2. TransportClient
3. Java High Level REST Client
4. Java Low Level REST Client  

以上4种方法各有千秋，它们之间的优劣不在此赘述，本文着重以3、4两种基于Rest Client的方式展开了解。  
***备注1：本文是基于ElasticSearch5.6系列，请酌情参考***  
***备注2：本文绝大部分内容代码都是本地验证后复制粘贴，如需更详细内容，请直接跳转'参考地址'***  
*[参考地址](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html) :https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html*  
*预备知识:*  
* Java、maven、elasticsearch基础概念(索引、类型、文档等)


### 一、Java High Level REST Client

#### 1、maven依赖
```
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>elasticsearch-rest-high-level-client</artifactId>
      <version>5.6.9</version>
    </dependency>
```

为解决版本冲突，可能还需要引入如下build  
```
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals><goal>shade</goal></goals>
                    <configuration>
                        <relocations>
                            <relocation>
                                <pattern>org.apache.http</pattern>
                                <shadedPattern>hidden.org.apache.http</shadedPattern>
                            </relocation>
                            <relocation>
                                <pattern>org.apache.logging</pattern>
                                <shadedPattern>hidden.org.apache.logging</shadedPattern>
                            </relocation>
                            <relocation>
                                <pattern>org.apache.commons.codec</pattern>
                                <shadedPattern>hidden.org.apache.commons.codec</shadedPattern>
                            </relocation>
                            <relocation>
                                <pattern>org.apache.commons.logging</pattern>
                                <shadedPattern>hidden.org.apache.commons.logging</shadedPattern>
                            </relocation>
                        </relocations>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### 2、初始化
首先是构建**RestClient**
```java
RestClient lowLevelRestClient = RestClient.builder(
      new HttpHost("localhost", 9200, "http"),
      new HttpHost("localhost", 9201, "http")).build();
```
其中builder还可设置MaxRetryTimeout、FailureListener、RequestConfigCallback、HttpClientConfigCallback，具体请查看*[参考地址](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html)*
```java
RestHighLevelClient client = new RestHighLevelClient(lowLevelRestClient);
```
#### 3、支持的操作
RestHighLevelClient 支持如下几类API  
**Single document APIs 单文档操作**
* Index API
* Get API
* Delete API
* Update API

**Multi document APIs 多文档操作**
* Bulk API 批量API

**Search APIs 搜索**
* Search API
* Search Scroll API
* Clear Scroll API

**Miscellaneous APIs**
* Info API  

#### 4、示例
**Index**
* 构建Index 请求
```java
IndexRequest request = new IndexRequest(
      "posts",
      "doc",  
      "1");   
String jsonString = "{" +
      "\"user\":\"kimchy\"," +
      "\"postDate\":\"2013-01-30\"," +
      "\"message\":\"trying out Elasticsearch\"" +
      "}";
request.source(jsonString, XContentType.JSON);
```
其中request.source(),除Json字符串外，还支持Map、XContentBuilder 、Key-Value键值对.

* Index 请求选项(可选)
```java
request.routing("routing");
request.parent("parent");
request.timeout(TimeValue.timeValueSeconds(1)); 或者 request.timeout("1s");
request.setRefreshPolicy(WriteRequest.RefreshPolicy.WAIT_UNTIL); 或者 request.setRefreshPolicy("wait_for");
request.version(2);
request.versionType(VersionType.EXTERNAL);
request.opType(DocWriteRequest.OpType.CREATE); 或者 request.opType("create");
request.setPipeline("pipeline");
```
* 执行请求
```java
  //同步请求：
  IndexResponse indexResponse = client.index(request);
  //异步请求：
  client.indexAsync(request, new ActionListener<IndexResponse>() {
    @Override
    public void onResponse(IndexResponse indexResponse) {}

    @Override
    public void onFailure(Exception e) {}
  });
```
* 结果处理(示例)
```java
  String index = indexResponse.getIndex();
  String type = indexResponse.getType();
  String id = indexResponse.getId();
  long version = indexResponse.getVersion();
  if (indexResponse.getResult() == DocWriteResponse.Result.CREATED) {

  } else if (indexResponse.getResult() == DocWriteResponse.Result.UPDATED) {

  }
  ReplicationResponse.ShardInfo shardInfo = indexResponse.getShardInfo();
  if (shardInfo.getTotal() != shardInfo.getSuccessful()) {

  }
  if (shardInfo.getFailed() > 0) {
      for (ReplicationResponse.ShardInfo.Failure failure : shardInfo.getFailures()) {
          String reason = failure.reason();
      }
  }
```
**GET**
* 构建Get 请求
```java
  GetRequest getRequest = new GetRequest("posts","doc", "1");
  //其中posts -> _index 索引; doc -> _type类型; 1 -> _id 文档ID
```  
* GET 请求选项(可选)
```java
  request.fetchSourceContext(new FetchSourceContext(false));
  String[] includes = new String[]{"message", "*Date"};
  String[] excludes = Strings.EMPTY_ARRAY;
  FetchSourceContext fetchSourceContext = new FetchSourceContext(true, includes, excludes);
  request.fetchSourceContext(fetchSourceContext);
  String[] includes = Strings.EMPTY_ARRAY;
  String[] excludes = new String[]{"message"};
  FetchSourceContext fetchSourceContext = new FetchSourceContext(true, includes, excludes);
  request.fetchSourceContext(fetchSourceContext);
  request.storedFields("message");
  request.routing("routing");
  request.parent("parent");
  request.preference("preference");
  request.realtime(false);
  request.refresh(true);
  request.version(2);
  request.versionType(VersionType.EXTERNAL);
```  
* GET执行请求
```java
  //同步请求(可能的异常处理)：
  GetRequest request = new GetRequest("does_not_exist", "doc", "1").version(2);
  try {
      GetResponse getResponse = client.get(request);
  } catch (ElasticsearchException e) {
      if (e.status() == RestStatus.NOT_FOUND) {
          //TODO
      }
      if (exception.status() == RestStatus.CONFLICT) {
        //TODO
    }
  }
  // 异步请求：
  client.getAsync(request, new ActionListener<GetResponse>() {
    @Override
    public void onResponse(GetResponse getResponse) {}

    @Override
    public void onFailure(Exception e) {}
    });
```  
* 结果处理(示例)
```java
  String index = getResponse.getIndex();
  String type = getResponse.getType();
  String id = getResponse.getId();
  if (getResponse.isExists()) {
    long version = getResponse.getVersion();
    String sourceAsString = getResponse.getSourceAsString();        
    Map<String, Object> sourceAsMap = getResponse.getSourceAsMap();
    byte[] sourceAsBytes = getResponse.getSourceAsBytes();          
  } else {
    //TODO
  }
```
**Others**
其他剩余内容请直接参考[原文](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html)，内容结构与本文一致
### 二、Java Low Level REST Client
Java High Level REST Client能适应绝大多数场景，如无法解决，可继续采用Java Low Level REST Client。Low Level 与 High Level 最大的不同在于Low 只有最简单的http封装，完全模拟curl 请求。
#### 1、maven 依赖

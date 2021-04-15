* Docker启动命令

```shell
docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" 8pf2fpd3.mirror.aliyuncs.com/library/elasticsearch:6.8.7

```

如果需要外部ip能够访问elasticsearch还需要设置跨域，进入容器内部，修改elasticsearch.yml文件

```shell
$ docker exec -it 660dd399bd2e /bin/bash
# cd config/
# vi elasticsearch.yml

修改为：
cluster.name: "docker-cluster"
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
 
# minimum_master_nodes need to be explicitly set when bound on a public IP
# set to 1 to allow single node clusters
# Details: https://github.com/elastic/elasticsearch/pull/17288
discovery.zen.minimum_master_nodes: 1
 
保存即可
```

## Spring Boot整合ElasticSearch

在Pom文件中有如下一个依赖：

```xml
<!-- SpringBoot默认使用SpringData elasticsearch模块操作Elasticsearch -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

### SpringBoot默认支持两种技术来和ES进行交互

#### 1、Jest（默认不生效，需要导入jest工具包）

添加如下依赖：

```xml
<dependency>
    <groupId>io.searchbox</groupId>
    <artifactId>jest</artifactId>
    <version>6.3.1</version>
</dependency>
```

在application.properties文件中添加如下配置：

```properties
spring.elasticsearch.jest.uris=http://192.168.0.105:9200
```

然后就可以开始编写测试代码了

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringBoot03ElasticsearchApplicationTests {
	
    //SpringBoot自动配置的
    @Autowired
    JestClient jestClient;
    
    @Test
    public void contextLoads() {
        //1.给es中索引（保存）一个文档
        //article是自己写的一个bean
        Article article = new Article();
        article.setId(1);
        article.setTitle("sispence");
        article.setAuthor("maugham");
        article.setContent("The Moon and Sixpence");

        //构建一个索引功能
        Index index = new Index.Builder(article).index("wyjie").type("arti").build();

        try {
            //执行
            jestClient.execute(index);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    
    //使用jest进行搜索
    @Test
    public void search() {
        //查询表达式
        String json = "{\"query\":{\"match\":{\"content\":\"moon\"}}}";
        //构建搜索功能
        Search search = new Search.Builder(json).addIndex("wyjie").addType("arti").build();
        //执行
        try {
            SearchResult searchResult = jestClient.execute(search);
            System.out.println(searchResult.getSourceAsString());
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```



#### 2、SpringData ElasticSearch

配置了如下内容：

1. Client 节点信息：clusterNodes、clusterName
2. ElasticsearchTemplate 操作es
3. 编写一个ElasticsearchRepository的子接口来操作es
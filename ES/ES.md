<H1>ElasticSearch-SpringBoot学习笔记<H1>

# 一.基本概念

**Elasticsearch是近乎实时的搜索平台。这意味着从索引文档到可搜索到这段时间之间会有轻微的延迟（通常是一秒钟）。**

**ES可以理解为一个数据库，用来存放数据，是noSql的形式，他的查询速度非常快，但是插入速度比较慢，所以比较适合作为查询的数据库**



|   属性   |                             概念                             |
| :------: | :----------------------------------------------------------: |
| Document |        相当于一条存储的JSON数据,对应java中的类的实体         |
|   Type   |     相同Document数据类型（结构）的集合（新版本已经弃用）     |
|  Index   | 多个Type或者Document的集合,但是用来表示Document的集合更为合适，因为一般把相同数据结构的Document作为一个索引，这也是弃用Type的原因之一 |
|          |                                                              |

==低版本可以把Index理解为一个项目里面的数据，type理解为项目里面的实体类，Document理解为每一个实例对象，但是后来发现用Index来区分不同结构的Document更为方便，所以高版本已经去掉了type这一概念==

# 二. 安装ES，Kibana和中文分词器

## 		1.ES安装

​			**ES在官网下载需要的版本解压即可，无需配置，开箱即用，进入Bin目录运行启动文件即可启动**

## 		2.Kibana安装

​			**Kibana是一款可视化的操作工具，在官网下载对应版本解压即可，进入bin目录启动，打开浏览器在地址栏输入http://localhost:5601/即可**

## 		3.中文分词器安装

​			**https://github.com/medcl/elasticsearch-analysis-ik/releases下载与安装的ES版本相同的版本解压到ES的plugins目录即可，重新启动ES会自动加载plugins**

![image-20210414142320923](ES.assets/image-20210414142320923.png)

​		**在安装好分词器之后，可以在Kibana中打开Dev Tools检查分词效果**

```json
GET _analyze
{
"text":"钛合金狗眼",
"analyzer":"ik_max_word"
}
```

​		**这是使用标准分词的效果，对中文十分不友好，会把搜索词按照每个字分割**

​			<img src="ES.assets/image-20210413142114190-1618388679528.png" alt="image-20210413142114190"  />

​		**这是使用中文分词的效果**

​			<img src="ES.assets/image-20210413142223451.png" alt="image-20210413142223451"  />

​	==注意：如果没有下载IK分词器，第二种请求会报错==

​	==其中**ik_max_word**表示根据最细粒度拆分，比如比如会将“中华人民共和国人民大会堂”拆分为“中华人民共和国、中华人民、中华、华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。==

​	==还有对应的**ik_smart**表示最粗粒度拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂==

# 三. spring-data-elasticsearch

在spring项目中，可以使用**spring-data-elasticsearch**来操作es，其提供了增删改查的接口==ElasticsearchRepository<T, ID extends Serializable>==，其中T是对应的实体类型，ID是主键ID，我们创建一个Product类。

## 		1. 准备工作

### 				1）引入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>org.example</groupId>
    <artifactId>es</artifactId>
    <version>1.0-SNAPSHOT</version>
    <!--别摸🐟-->
    <dependencies>
        <!-- 工具类 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.8.1</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework.data/spring-data-elasticsearch -->
        <!-- es高版本需要使用spring-data-elasticsearch3.1.X相关依赖-->
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-elasticsearch</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.12</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-test -->
        <!--主要使用Test进行测试-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>


    </dependencies>
</project>
```

### 				2）创建Product产品类

```java
@Document(indexName = "product")
@Data
public class Product {
    @Id
    private int id;

    /**
     * 分词器 analyzer 的作用有二：
     *
     * 一是 插入文档时，将 text 类型字段做分词，然后插入 倒排索引。
     * 二是 在查询时，先对 text 类型输入做分词， 再去倒排索引搜索。
     * 如果想要“索引”和“查询”， 使用不同的分词器，那么 只需要在字段上
     * 使用 search_analyzer。这样，索引只看 analyzer，查询就看 search_analyzer。
     * 两种 分词器的最佳实践： 索引时用 ik_max_word（面面俱到）， 搜索时用 ik_smart（精准匹配）
     */

    /**
     * Field的Type类型分为以下几类
     * 1，String 类，分为两种：
     *
     * text：可分词，不参与聚合
     * keyword：不可分词，数据会作为完整字段进行匹配，可参与聚合
     *
     * 2，Numberical 数值类型，分两类：
     * 基本数据类型：long、integer、short、byte、double、float、half_float
     * 浮点数高精度类型：scaled_float（需要制定精度因子，10或100这样，es会把真实值与之相乘后存储，取出时还原）
     *
     * 3，Date 日期类型
     * ES 可以对日期格式，化为字符串存储，但是我们建议存储为毫秒值 long，节省空间
     */
    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String name;

    private String price;
    
}
```

+ 相关注解详解

  -  **@Document**

  ```java
  public @interface Document {
  
  	String indexName(); //索引库的名称，个人建议以项目的名称命名，高版本建议以类名命名
  
  	String type() default ""; //类型，个人建议以实体的名称命名,高版本已经删除，用IndexName代替更为合适
  
      /**
       * 表示当Spring创建索引时,Spring不会在创建的索引中设置以下设置：shards,replicas,refreshInterval和indexStoreType.
       * 这些设置将是Elasticsearch默认值(服务器配置)
       */
  	boolean useServerConfiguration() default false;
  
  	short shards() default 5;//默认分区数
  
  	short replicas() default 1;//每个分区默认的备份数
  
  	String refreshInterval() default "1s";//刷新间隔
  
  	String indexStoreType() default "fs";//索引文件存储类型
  
  	boolean createIndex() default true; //表示当Spring应用程序启动时,如果配置的索引不存在,则Spring会创建索引
  }
  
  ```

  

  - **@Field**

  ```java
  public @interface Field {
  
  	FieldType type() default FieldType.Auto;//自动检测属性的类型，可以根据实际情况自己设置
  
  	boolean index() default true;//是否建立倒排索引
  
  	DateFormat format() default DateFormat.none; //时间类型的格式化
  
  	String pattern() default ""; //时间类型格式化匹配模式
  
  	boolean store() default false;//默认情况下不存储原文
  
  	boolean fielddata() default false;
  
  	String searchAnalyzer() default "";//指定字段搜索时使用的分词器
  
  	String analyzer() default "";//指定字段建立索引时指定的分词器
  
  	String normalizer() default "";
  
  	String[] ignoreFields() default {};//如果某个字段需要被忽略
  
  	boolean includeInParent() default false;
  
  	String[] copyTo() default {};
  }
  ```

### 				3）创建ProductEsDao接口

​			创建ProductEsDao接口并继承ElasticsearchRepository<Product, Integer>，相关的CRUD操作都在这个接口里面

```java
@Component
public interface ProductEsDao extends ElasticsearchRepository<Product, Integer> {

}
```

### 				4）创建一个测试类

​			用单元测试比较方便

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Run.class)
public class ProductEsServiceTest {
    @Autowired
    private ProductEsDao productEsDao;
}
```



## 	2.接口关系

​		

+ CRUD接口全部继承**interface Repository<T, ID>**这个顶层接口，其中**CrudRepository<T, ID>**是子接口,其源码如下

```java
@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {

	//保存（更改或者新增）给定的entity
	<S extends T> S save(S entity);

	//保存（更改或者新增）给定的entities
	<S extends T> Iterable<S> saveAll(Iterable<S> entities);

	//通过主键Id查新
	Optional<T> findById(ID id);

	//查询是否存在
	boolean existsById(ID id);

	//查询所有
	Iterable<T> findAll();

	//通过Ids查询所有
	Iterable<T> findAllById(Iterable<ID> ids);

	//返回所用实体的数量
	long count();

	//删除该Id的数据
	void deleteById(ID id);

	//删除指定entity
	void delete(T entity);

	//删除指定entities
	void deleteAll(Iterable<? extends T> entities);

	//删除所有该Repository所管理的所有数据
	void deleteAll();
}
```

+ **PagingAndSortingRepository<T, ID>**继承自**CrudRepository<T, ID>**，其源码如下，提供了排序和分页功能

```java
@NoRepositoryBean
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {

	//传入排序参数，对查询结果进行排序
	Iterable<T> findAll(Sort sort);

	//传入分页参数，将查询结果包装成分页形式
	Page<T> findAll(Pageable pageable);
}

```

+  **ElasticsearchCrudRepository<T, ID extends Serializable>**继承**PagingAndSortingRepository<T, ID>**,未提供任何方法， **ElasticsearchRepository<T, ID extends Serializable>**继承 **ElasticsearchCrudRepository<T, ID extends Serializable>**，主要的操作都是使用这个接口，，这个接口增对ES提供了多种操作，其源码如下

```java
@NoRepositoryBean
public interface ElasticsearchRepository<T, ID extends Serializable> extends ElasticsearchCrudRepository<T, ID> {
	//建立索引，底层调用了save()方法，save()方法会对不存在的索引新增
	<S extends T> S index(S entity);
	//查询所有，传入参数是QueryBuilder，这个类型后续详细讲解，返回不做处理的结果集
	Iterable<T> search(QueryBuilder query);
	//根据分页信息查询，返回分页集合
	Page<T> search(QueryBuilder query, Pageable pageable);
	//查询所有，传入参数是SearchQuery，这个类型后续详细讲解，返回不做处理的结果集
	Page<T> search(SearchQuery searchQuery);
	//使用morelikethis查询搜索相似的实体
	Page<T> searchSimilar(T entity, String[] fields, Pageable pageable);
	//刷新Index
	void refresh();
	//获取该Repository所管理的实体类型
	Class<T> getEntityClass();
}
```



## 	3.AbstractElasticsearchRepository

​			该类是 **spring-data-elasticsearch**提供的一个**ElasticsearchRepository**实现类，里面提供了已经实现了**ElasticsearchRepository**的CRUD方法，也提供了一些其他自己的方法，可满足大部分需求，其实现原理是使用了ElasticsearchOperations操作类，通过调用其方法实现**ElasticsearchRepository**的CRUD

其部分源码如下：

​	

```java
public abstract class AbstractElasticsearchRepository<T, ID extends Serializable>
		implements ElasticsearchRepository<T, ID> {

	static final Logger LOGGER = LoggerFactory.getLogger(AbstractElasticsearchRepository.class);
    //这个是spring提供的一个封装好的一个关于ES的操作类，CRUD方法都是调用这个操作类里面的方法
	protected ElasticsearchOperations elasticsearchOperations;
	protected Class<T> entityClass;
    //实体的元信息
	protected ElasticsearchEntityInformation<T, ID> entityInformation;

    /**
   	构造函数省略
    **/

    //可以看到下面的方法大部分都是封装了elasticsearchOperations的操作，也就是说主要的操作都是elasticsearchOperations来完成
	private void createIndex() {
		elasticsearchOperations.createIndex(getEntityClass());
	}

	private void putMapping() {
		elasticsearchOperations.putMapping(getEntityClass());
	}

	private boolean createIndexAndMapping() {
		return elasticsearchOperations.getPersistentEntityFor(getEntityClass()).isCreateIndexAndMapping();
	}

	@Override
	public Optional<T> findById(ID id) {
		GetQuery query = new GetQuery();
		query.setId(stringIdRepresentation(id));
		return Optional.ofNullable(elasticsearchOperations.queryForObject(query, getEntityClass()));
	}

	@Override
	public Iterable<T> findAll() {
		int itemCount = (int) this.count();
		if (itemCount == 0) {
			return new PageImpl<>(Collections.<T> emptyList());
		}
		return this.findAll(PageRequest.of(0, Math.max(1, itemCount)));
	}

	/**
	 **其他方法省略，详细可以查看org.springframework.data.elasticsearch.repository.support.AbstractElasticsearchRepository<T, ID 	 **extends Serializable>;
	 **/
}

```



## 	4.ElasticsearchOperations和ElasticsearchTemplate

​		**ElasticsearchTemplate**是ElasticsearchOperations的具体实现，也就是说数据操作的具体实现需要参考**ElasticsearchTemplate**模板，部分源码如下

​		

```java
	@Override
	public <T> AggregatedPage<T> queryForPage(SearchQuery query, Class<T> clazz, SearchResultMapper mapper) {
		SearchResponse response = doSearch(prepareSearch(query, clazz), query);
		return mapper.mapResults(response, clazz, query.getPageable());
	}
```

==其中**SearchResultMapper**是一个接口，采用了模板方法的设计模式，需要传入参数的时候进行方法的具体实现==

# 四.操作案例

## 		1.增加（更新）操作

### 				1）增加单条记录

```java
@Test
public void saveTest() {
    
    product = new Product();
    product.setId(34123);
    product.setName("笔记本");
    product.setPrice("123");
    productEsService.save(product);
}
```

==第一次新增是如果不存在Product类上指定的索引，则会自动新建索引==

### 				2）增加多条记录

```java
    @Test
    public void saveAllTest() {
        List list = new LinkedList<Product>();
        list.add(new Product(1, "小米手机", "145"));
        list.add(new Product(2, "小米", "200"));
        list.add(new Product(3, "脸盆", "130"));
        list.add(new Product(4, "洗脸夜", "130"));
        list.add(new Product(5, "衣架", "130"));
        list.add(new Product(6, "洗衣液", "130"));
        list.add(new Product(7, "奥特曼手办", "130"));
        list.add(new Product(8, "小米手机手办", "130"));
        list.add(new Product(9, "钛合金狗眼", "130"));
        list.add(new Product(10, "合金大默器", "130"));
        list.add(new Product(11, "中华人民共和国", "130"));
        list.add(new Product(12, "何锦康穿过的袜子", "130"));
        list.add(new Product(13, "何锦康放的屁", "130"));
        list.add(new Product(14, "Java高并发编程详解", "130"));
        list.add(new Product(15, "Maven实战教程", "130"));
        list.add(new Product(16, "小吉他", "130"));
        list.add(new Product(17, "高级开发实战", "130"));
        list.add(new Product(18, "羽绒衣服", "130"));
        productEsDao.saveAll(list);
    }
```



==增加更新都是使用save()方法，根据主键决定是更新（覆盖）还是新增==

## 	2.删除操作

```java
    @Test
    public void deleteTest() {
        productEsService.delete(product);

    }

    @Test
    public void deleteAll() {
        //删除指定索引（Product类上的@Document的属性指定）的全部数据
        productEsDao.deleteAll();
        
        //删除集合里面的数据
        productEsDao.deleteAll(List<Product> products);
    }

```

## 	3.查询操作

### 		1）查询器

+  在做查询操作的时候一般都是调用**ElasticsearchRepository**的**search(SearchQuery searchQuery)**方法，这个方法里面的参数是一个接口，接口关系如下图所示：

  ![SearchQuery](ES.assets/SearchQuery.jpg)

+  可以看到我们构建**NativeSearchQuery**来完成一些复杂的查询

==一般情况下，我们不是直接是new NativeSearchQuery，而是使用NativeSearchQueryBuilder。
通过NativeSearchQueryBuilder.withQuery(QueryBuilder1).withFilter(QueryBuilder2).withSort(SortBuilder1).withXXXX().build();这样的方式来完成NativeSearchQuery的构建。==

### 		2）查询器构建

```java
   @Test
    public void queryBuilderTest() {
        String searchWord = "小米";
        //查询条件构造,查询name含有小米的数据
        QueryBuilder queryBuilder = QueryBuilders.matchQuery("name", searchWord);
        //结果排序构造  @Field(type = FieldType.Text) 类型的无法进行聚合操作，所以这里使用Id
        SortBuilder sortBuilder = SortBuilders.fieldSort("id");
        //分页查询构造，注意这里是从0开始的
        Pageable pageable = PageRequest.of(0, 3);

        //构建一个SearchQuery
        SearchQuery searchQuery =
                new NativeSearchQueryBuilder()
                        .withQuery(queryBuilder)
                        .withSort(sortBuilder)
                        .withPageable(pageable)
                        .build();
        /*
        通过调用Search方法，传入SearchQuery接口类型的参数，返回一个分页集合，再调用Page的 getContent()获取结果集
        */
        Page<Product> search = productEsDao.search(searchQuery);
        System.out.println("查询词汇是" + searchWord);
        System.out.println("共" + search.getTotalElements() + "条结果集" + search.getContent());
    }
```

![image-20210414142403863](ES.assets/image-20210414142403863.png)

### 		3）termQuery

```java
@Test
public void termQueryTest() {
    //查询结果为空，包含查询，数据库不存在包含小米手办的数据
    QueryBuilder queryBuilder = QueryBuilders.termQuery("name", "小米手办");
    SearchQuery searchQuery =
        new NativeSearchQueryBuilder()
        .withQuery(queryBuilder)
        .build();
    Page<Product> search = productEsDao.search(searchQuery);
    System.out.println("共" + search.getTotalElements() + "条结果集" + search.getContent());
}
```

### 		4）matchQuery

```java
@Test
public void matchQueryTest() {
    //共5条结果集[Product(id=8, name=小米手机手办, price=170),
    // Product(id=7, name=奥特曼手办, price=160),
    // Product(id=2, name=小米, price=110),
    // Product(id=34, name=小米手环, price=346),
    // Product(id=1, name=小米手机, price=100)]
    //拆分为小米，手办进行匹配查询
    QueryBuilder queryBuilder = QueryBuilders.matchQuery("name", "小米手办");
    SearchQuery searchQuery =
            new NativeSearchQueryBuilder()
                    .withQuery(queryBuilder)
                    .build();
    Page<Product> search = productEsDao.search(searchQuery);
    System.out.println("共" + search.getTotalElements() + "条结果集" + search.getContent());
}
```

### 		5）rangeQuery

```java
@Test
public void rangeQueryTest() {
    //共6条结果集[Product(id=5, name=衣架, price=140),
    // Product(id=8, name=小米手机手办, price=170),
    // Product(id=9, name=钛合金狗眼, price=180),
    // Product(id=4, name=洗脸夜, price=130),
    // Product(id=6, name=洗衣液, price=150),
    // Product(id=7, name=奥特曼手办, price=160)]
    //包含范围边界 是否包含边界可以使用第二个参数进行设定 或者使用
    // gt = from 不包含起始边界 lt = to 不包含起始边界
    // gte lte 与上面相反
    QueryBuilder queryBuilder = QueryBuilders.rangeQuery("price").from(130, true).to(180,true);

    SearchQuery searchQuery =
        new NativeSearchQueryBuilder()
        .withQuery(queryBuilder)
        .build();
    Page<Product> search = productEsDao.search(searchQuery);
    System.out.println("共" + search.getTotalElements() + "条结果集" + search.getContent());
}
```

### 		6）boolQuery

```java
@Test
public void boolQueryTest() {

    //多构造器组合查询，must，mustNot等等，取交集
    QueryBuilder queryBuilder1 = QueryBuilders.rangeQuery("price").from(130, true).to(180,true);
    QueryBuilder queryBuilder2 = QueryBuilders.matchQuery("name", "华为手机");
    //共1条结果集[Product(id=8, name=小米手机手办, price=170)]
    QueryBuilder queryBuilder = QueryBuilders.boolQuery().must(queryBuilder1).must(queryBuilder2);
   
    SearchQuery searchQuery =
            new NativeSearchQueryBuilder()
                    .withQuery(queryBuilder)
                    .build();
    Page<Product> search = productEsDao.search(searchQuery);
    System.out.println("共" + search.getTotalElements() + "条结果集" + search.getContent());
}
```

### 		7）自定义查询

+  可以使用方法名来实现查询基本功能，不需要实现代码，方法名需要按照规范定义，比如我实现一个根据id和姓名查询实体，则需要定义方法名为**queryProductByIdAndName(int id, String name)**，参数根据实际情况填写，不需要实现既可以进行查询

  ```java
  @Component
  public interface ProductEsDao extends ElasticsearchRepository<Product, Integer> {
  
      Product queryProductByIdAndName(int id, String name);
  
      List<Product> queryProductByIdOrName(int id, String name);
  
      Product queryProductById(int id);
  
      Product queryProductByName(String name);
  }
  
  ```

  ```java
  @Test
  public void customerQuery() {
      //自定义方法名，不需要实现，只需用query（查询）Product（实体）By(通过)Id(属性)And(并且)Name(属性)
      System.out.println(productEsDao.queryProductByIdOrName(8, "手机"));
      System.out.println(productEsDao.queryProductById(8));
      System.out.println(productEsDao.queryProductByName("衣架"));
      System.out.println(productEsDao.queryProductByIdAndName(19, "水杯"));
  }
  ```

  ![image-20210414160748825](ES.assets/image-20210414160748825.png)

  +  更多详细的用法请参考下图

  ![image-20210414155251591](ES.assets/image-20210414155251591.png)

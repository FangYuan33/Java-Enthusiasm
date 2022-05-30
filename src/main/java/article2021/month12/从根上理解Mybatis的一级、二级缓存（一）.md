
## 1. 写在前头
大家好，我是`方圆`，最近在读Mybatis的源码，之前面试被问过一、二级缓存相关的问题，想着就把这写成一篇博客记录下来吧，供回忆，供参考

这篇帖子主要讲`一级缓存，它的作用范围和源码分析`

（本来想把一二级缓存合在一起，发现太长了）

---
## 2. 准备工作
### 2.1 两个要用的实体类

```java
public class Department {

    public Department(String id) {
        this.id = id;
    }

    private String id;

    /**
     * 部门名称
     */
    private String name;

    /**
     * 部门电话
     */
    private String tel;

    /**
     * 部门成员
     */
    private Set<User> users;
}
```

```java
public class User {

    private String id;

    private String name;

    private Integer age;

    private LocalDateTime birthday;

    private Department department;
}
```

### 2.2 Mapper.xml文件中要用的SQL
- DepartmentMapper.xml，两条SQL，一条根据ID匹配，一条清除缓存，注意`fulshCache`标签

```xml
    <select id="findById" resultType="Department">
        select * from department
        where id = #{id}
    </select>

    <!-- flushCache 所有namespace 的一级缓存 和 当前namespace 的二级缓存均会清除 默认是false-->
    <select id="cleanCathe" resultType="int" flushCache="true">
        select count(department.id) from department;
    </select>
```
- UserMapper.xml，简简单单的查询所有的user

```java
    <select id="findAll" resultMap="userMap">
        select u.*, td.id, td.name as department_name
        from user u
        left join department td
        on u.department_id = td.id
    </select>
```

---
## 3. 一级缓存
- 一级缓存是基于SQLSession的，同一条SQL执行第二遍的时候会直接从缓存中取
  测试下看看

```java
    public static void main(String[] args) throws IOException {
        InputStream xml = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
        // 开启二级缓存需要在同一个SqlSessionFactory下，二级缓存存在于 SqlSessionFactory 生命周期，如此才能命中二级缓存
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(xml);

        SqlSession sqlSession = sqlSessionFactory.openSession();
        DepartmentMapper departmentMapper = sqlSession.getMapper(DepartmentMapper.class);
        System.out.println("----------department第一次查询 ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println("----------department一级缓存生效，控制台看不见SQL ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");

    }
```

- 可以发现控制台在第二次查询的时候，一级缓存生效，没有出现SQL
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/4743d1d257c642199d69e0c87f08b021.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 我们清空下一级缓存再试试

> xml文件中`flushCache标签` 会清除所有namespace 的一级缓存 和 当前namespace 的二级缓存均会清除 默认是false

```java
    public static void main(String[] args) throws IOException {
        InputStream xml = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
        // 开启二级缓存需要在同一个SqlSessionFactory下，二级缓存存在于 SqlSessionFactory 生命周期，如此才能命中二级缓存
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(xml);

        SqlSession sqlSession = sqlSessionFactory.openSession();
        DepartmentMapper departmentMapper = sqlSession.getMapper(DepartmentMapper.class);
        System.out.println("----------department第一次查询 ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println("----------department一级缓存生效，控制台看不见SQL ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println("----------清除一级缓存 ↓------------");
        departmentMapper.cleanCathe();
        System.out.println("----------清除后department再一次查询，SQL再次出现 ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
    }
```
- 控制台日志很清晰，清除缓存后又重新查了一遍
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/c6db55d579d34ccfb66817dc513268c6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
### 3.1 一级缓存失效的情况
#### 3.1.1 不同SQLSession下同一条SQL一级缓存不生效
- 创建一个新的sqlSession1执行相同的SQL，发现不同SQLSession下不共享一级缓存

```java
        SqlSession sqlSession = sqlSessionFactory.openSession();
        SqlSession sqlSession1 = sqlSessionFactory.openSession();
        DepartmentMapper departmentMapper = sqlSession.getMapper(DepartmentMapper.class);
        DepartmentMapper departmentMapper1 = sqlSession1.getMapper(DepartmentMapper.class);
        System.out.println("----------department第一次查询 ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println("----------sqlSession1下department执行相同的SQL，控制台出现SQL ↓------------");
        departmentMapper1.findById("18ec781fbefd727923b0d35740b177ab");
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/f84b8886001144eea039dbf753c48fef.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 3.1.2 两次相同查询SQL间有Insert、Delete、Update语句出现

- 因为Insert、Delete、Update的`flushCache标签 默认为 true` ，执行它们时，必然会导致一级缓存的清空，从而引发之前的一级缓存不能继续使用的情况（这跟我们上边清除一级缓存的SQL例子一致）

#### 3.1.3 调用sqlSession.clearCache()方法

- 这个方法会将一级缓存清除，效果是一样的

### 3.2 一级缓存源码：缓存被保存在了哪里？
#### 3.2.1 该如何找打它的位置
- Mybatis顶层的缓存是`接口Cache`，查看它的实现类
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/001462d52f6d4e9280a03b07516b10a3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
发现大部分实现类的包都是`decorators(装饰器)`，只有PerpetualCache是Impl，所以我们确定的说，它就是我们要找的缓存实现类，点进去看看，发现只是组合了HashMap...

```java
public class PerpetualCache implements Cache {

  private final String id;

  // 看这里
  private final Map<Object, Object> cache = new HashMap<>();

  ...
}
```

- 那这个PerpetualCache被放在哪里呢？
  我们想到了一级缓存是`基于SQLSession`，那我们去DefaultSQLSession，它默认的实现类里看看

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  ...
}
```
- 发现并没有哇！DefaultSqlSession还有两个东西，Configuration是全局的配置，这里边儿应该是没有，那我们只能`再去Executor里`看看了

![在这里插入图片描述](https://img-blog.csdnimg.cn/63a07b2d1091450ea044aa54cb792023.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
-  发现它是个接口，实现类有一个`CachingExecutor`！立马点进去！

```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  ...
}
```
- 发现还是没有？？？
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/7de7a4ed074c4459b3c99b6c611467d5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_10,color_FFFFFF,t_70,g_se,x_16)
- 但是Executor还有一个`BaseExecutor`，最后一家了，再在没有关了Idea睡觉了

```java
public abstract class BaseExecutor implements Executor {

  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  protected Transaction transaction;
  protected Executor wrapper;

  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  // o??!! 不就在这呢嘛，小老弟
  protected PerpetualCache localCache;
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;

  ...
}
```
- 它来了，原来在这藏着呢呀，行了，这把知道它的位置了，我们直接看SQL执行的时候是怎么存的，怎么取的吧！

#### 3.2.2 query()方法
- BaseExecutor的query()方法，看看注释，很简单

```java
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 是否需要清除一级缓存
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 查询一级缓存中是否存在数据
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        // 有数据直接取一级缓存
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 没有数据则去数据库中查
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      deferredLoads.clear();
      // 全局localCacheScope设置为statement，则清空一级缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        clearLocalCache();
      }
    }
    return list;
  }
```

#### 3.2.3 写两条Sql，Debug看一下

```java
        System.out.println("----------department第一次查询 ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println("----------department一级缓存生效，控制台看不见SQL ↓------------");
        departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
```

- 哎，很对，第一次果然去数据库里查了
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/74644ed2cbd741a8afc298757a0f854c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 哎，更对了，第二次果然取得缓存
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/cb52ae76c1ac4753be8f4380fb2aec04.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 好嘛，真简单呀

### 3.3 注意：一级缓存的查询结果被修改后，竟然...
- 竟然会对之后取出的一级缓存有影响，测试下看看

```java
        System.out.println("----------department第一次查询 ↓------------");
        Department department = departmentMapper.findById("18ec781fbefd727923b0d35740b177ab");
        System.out.println(department);
        department.setName("方圆把名字改了");

        System.out.println("----------department一级缓存生效，控制台看不见SQL ↓------------");
        System.out.println(departmentMapper.findById("18ec781fbefd727923b0d35740b177ab"));
```

- 第一次查询结果name为null，之后我们修改它的name，第二次查询取缓存的结果`是更改name结果之后的`
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/8761d870f7ec4d72960f0680f20c9157.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 这是因为`存放的数据其实是对象的引用`，导致第二次从一级缓存中查询到的数据，就是我们刚刚改过的数据


### 3.4 文末
一级缓存到这里就要跟大家说再见了，做个总结吧

- 一级缓存是基于SQLSession的，不同SQLSession间不共享一级缓存
- 执行Insert、Delete、Update语句会使一级缓存失效
- 一级缓存在底层被存放在了BaseExecutor中，本质上就是个HashMap
- 一级缓存存放的数据其实是对象的引用，若对它进行修改，则之后取出的缓存为修改后的数据

---
## 巨人的肩膀
- [ 为什么要实现序列化：MyBatis的一级缓存、二级缓存演示以及讲解，序列化异常的处理](https://www.cnblogs.com/ncl-960301-success/p/10976255.html)

- [为什么MyBatis二级缓存Cache Hit Ratio始终等于0：二级缓存的生命周期在同一个SqlSessionFactory中](https://www.it610.com/article/1304085366302609408.htm)
- 《玩转 MyBatis：深度解析与定制》第十四章

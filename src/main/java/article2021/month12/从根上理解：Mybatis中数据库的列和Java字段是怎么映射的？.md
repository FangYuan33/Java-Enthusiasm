
## 1. 写在前头

我们使用Mybatis时要写 `mapper.xml `，因为MyBatis 不像 Hibernate 那样是全自动 ORM ，对于实体类型它无法直接识别，这里面我们要自己定义 `resultMap ，手动实现映射`。

那它在底层是`如何实现将实体类的字段和数据库的列一一对应`的呢？这也是我小米二面的一道面试题：**“你知道Mybatis中数据库的列和Java实体类是怎么对应上的吗？”**

那我们就从浅入深学习一下吧，我看看谁还能问的倒我！

**不想看具体源码直接拖到最后看答案**（那就亏大啦 /doge)
![在这里插入图片描述](https://img-blog.csdnimg.cn/0b8dd57c8234454db346baadef958f02.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_7,color_FFFFFF,t_70,g_se,x_16)



---
## 2. 准备工作
- 创建一个供我们映射的Java实体类，非常的简单只是`id，name，tel`三个字段

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

}
```
- 创建DepartmentMapper.xml，定义ResultMap将字段和数据库列一一映射

```xml
 <resultMap id="lazyDepartmentMap" type="entity.Department">
     <id property="id" column="id"/>
     <result property="name" column="name"/>
     <result property="tel" column="tel"/>
 </resultMap>
```
- 注意在mybatis的配置文件mapper标签中要标记DepartmentMapper.xml

```xml
<mappers>
    <mapper resource="mapper/DepartmentMapper.xml"/>
</mappers>
```

**准备好了，我们步入正题！**

---
## 3.  加载配置文件中mappers标签

> `注意`：因为本次我们只讲映射的实现，所以我会将无关的源码都省去，因为不是逐条分析的源码可能会有些跳跃，但是逐条分析又显得不针对问题，所以我**建议大家结合文章内容自己Debug一下**，我觉得这样才是最好的状态，大家注意看`代码中的注释`，我会将需要注意的点都标记出来

- mybatis在启动的时候，会调用`parseConfiguration方法`，其中需要关注`mapperElement方法`，这是对我们指定的mapper.xml文件进行加载的过程，下方源码大家扫一眼，重点关注标记的最后一句
```java
  private void parseConfiguration(XNode root) {
    try {
      // 这里就是对配置文件的加载，大家看其中的字符串是不是和我们在配置文件中写的标签一致
      // properties, settings, typeAliases...这些，重点需要关注最后一个mappers
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      
      // 加载配置文件中我们指定的mappers，这里对应我们上边指定的DepartmentMapper.xml文件
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
### 3.1 mapperElement方法，加载mapper.xml
- 加载mapper.xml文件时，有package包扫描和根据resource, url, mapperClass指定扫描，下面源码中也标记的很清楚。因为我们在本例中使用的是`resource`，所以我就把无关的代码擦掉啦

```java
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
      	// 包扫描
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          // 三种不同的指定方式 resource url class 
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          // resource加载方式
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());

			// 这里我们要进入正题了奥！
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            // url加载 ...
          } else if (resource == null && url == null && mapperClass != null) {
            // class加载 ...
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```
### 3.2 parse方法，加载mapper.xml的具体步骤

-  核心的方法步骤都在这里了，我们重点看标记了数字的代码行

```java
  public void parse() {
    // 判断mapper.xml文件是否被加载过
    if (!configuration.isResourceLoaded(resource)) {
      // 1. 加载mapper元素 
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      // 绑定命名空间，这个是不是很熟悉？ 
      bindMapperForNamespace();
    }

	// 2. 解析resultMap
    parsePendingResultMaps();
    // 解析cache-ref，与二级缓存有关
    parsePendingCacheRefs();
    // 解析statement，statement是我们写的select等SQL代码
    parsePendingStatements();
  }
```
我们先看`configurationElement方法`

#### 3.2.1 加载mapper元素，configurationElement方法
- configurationElement(parser.evalNode(`"/mapper"`));
  注意方法调用中有/mapper参数，这个其实是我们xml文件中最顶层的标签
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/26a1d080a3fe407281a04669fc982655.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 实现细节如下
```java
 private void configurationElement(XNode context) {
    try {
      // 获取命名空间
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.isEmpty()) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      // 二级缓存相关
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      // 官方已废弃的东西
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      
      // 这一步，它来了，解析resultMap！ 我们要关注它！
      resultMapElements(context.evalNodes("/mapper/resultMap"));

      // sql标签和我们写的SQL
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```
我们要深入`resultMapElements方法`，**抓好了，前边路抖！**

##### 3.2.1.1 解析resultMap，resultMapElements方法
- 多次调用resultMapElement方法

```java
  // 这里是一个for循环，因为我们会定义多个resultMap嘛
  private void resultMapElements(List<XNode> list) {
    for (XNode resultMapNode : list) {
      try {
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }
```
- 下面我们直接看`resultMapElement核心方法`，**有点儿长警告**！

![在这里插入图片描述](https://img-blog.csdnimg.cn/d3f0bef40b4440d4b42d5a6fd0576364.png)

**一步步来，没有我们看不完的代码！**

```java
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());

    // 解析映射目标对应的实体类类型，在我们的xml文件中使用的是type标签
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));

    // 加载出实体类类型
    Class<?> typeClass = resolveClass(type);
    if (typeClass == null) {
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<>(additionalResultMappings);
    
    // 解析resultMap的子标签，并封装成resultMapping
    // 子标签constructor，这个是指定构造器
    // 子标签discriminator，鉴别器，它有妙用，可以根据数据不同指定不同的resultMap，大家可以了解一下
    // id和result子标签，常用，也是我们xml文件中使用的标签
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        // 因为我们在xml文件中只指定了id和result，所以会直接跑到这里执行
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        // result标签直接注册成resultMapping，下面我贴了两张Debug图供大家加深理解
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    // 获取resultMap的id值，我们这里id是lazyDepartmentMap
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // 看看是否有继承的resultMap
    String extend = resultMapNode.getStringAttribute("extends");
    // autoMapping标签
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // 利用ResultMapResolver处理resultMap
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      // 最后一步
      return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```

- `buildResultMappingFromContext方法`我就不粘出来了，因为它只是简单的`取标签属性值`罢了，没什么意思，下面儿两张图供大家加深理解，在图之后我们再简单的看一下`最后一步`resultMapResolver.`resolve方法`
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/9e1fe0cbdeb14ea1aa1248ae8e4cfd1c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/e325305c93fb4b98b0aa33db3dfd54d8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 我把最后一步代码抽出来，方便大家看

```java
    try {
      // 最后一步
      return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
```

- 这里调用的是`MapperBuilderAssistant的resolve方法`，它会把生成的resultMap封装到`Configuration`的`resultMaps`中

```java
  private final MapperBuilderAssistant assistant;

  public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
  }

  // 封装到Configuration的resultMaps中
  public ResultMap addResultMap(...) {
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
      ...
    }
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    // 这里，便加入到了configuration的resultMaps中
    configuration.addResultMap(resultMap);
    return resultMap;
  }
```
- Configuration中的resultMaps字段
```java
protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
```
#### 3.2.2 我们回到parse方法中，再看parsePendingResultMaps方法

```java
  private void parsePendingResultMaps() {
  
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    synchronized (incompleteResultMaps) {
      Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
      while (iter.hasNext()) {
        try {
          // 逐个解析
          iter.next().resolve();
          iter.remove();
        } catch (IncompleteElementException e) {
          // ResultMap is still missing a resource...
        }
      }
    }
  }
```
- 这里取出来`一组ResultMapResolver`，注意它有个单词`incomplete`，不完整的，说明来到这里的`resultMap还没有真正的解析完成`，需要在这里继续解析，而pending又有悬而未决的意思，现在觉得能`自解释`的代码真的很酷
- 因为我们定义的resultMap是已经完整解析的，我们Debug到这里incompleteResultMaps的大小为0，不必继续解析

- 那什么样的是没有完整解析的呢？我们需要再看看3.2.1.1节中`resultMapElement方法`最末尾的代码

```java
    try {
      return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
      // 这里添加了IncompleteResultMap
      // 说明上边的resolve方法必须抛出IncompleteElementException异常才行
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
```

那**什么情况下**能抛出`IncompleteElementException异常`？我们需要再看一下`resultMapResolver.resolve方法`中的代码

```java
  public ResultMap addResultMap(...) {
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
      // 这里，resultMap中必须指定了extend标签
      // 且当前configuration中不包含我们继承的resultMap，就会抛出这个异常
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
		......
  }
```

在有继承的resultMap时，也就是`result指定了extend标签`，但是这个被继承的resultMap`还没`被解析完成：`Configuration中还没有这个ResultMap`，这时就会抛出这个异常，那么就会被添加到IncompleteResultMap中

- 我们试一下，给我们定义的resultMap添加上extend标签，如下

```xml
    <resultMap id="lazyDepartmentMap" type="entity.Department" 
    extends="dao.UserMapper.userMap">
    
        <id property="id" column="id"/>
        <result property="name" column="name"/>
        <result property="tel" column="tel"/>
    </resultMap>
```
- Debug一下，确实抛出了这个异常，进入到了这里
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/928deef933a8452bb1887b95ea83c6a8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 我们再回到`parsePendingResultMaps方法`看看
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/c1bad6c9da414062968f94a0190338bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
  这下进来的正是我们定义的这个`没完全解析的ResultMap`，需要进一步完成解析才行，好了，到这里ResultMap的加载就说完了

---

## 4. 我们该怎么回答这个问题
Java实体类与数据库列`在mybatis中是半自动ORM映射`，需要我们`指定ResultMap`，`将实体类的字段和数据库列一一对应`，在底层中它解析成`resultMapping`的形式，通过`ResultMapResolver生成对应的ResultMap`

---
## 5. 写在最后
这一部分源码读起来很清晰也很简单，也是因为我们只是从`面上`读了源码，并没有深入到它的具体细节中，包括生命周期什么的，我想如果在一篇帖子中写完那估计得上万字才行了，所以我把深入理解的放在新的帖子中吧

---
## 巨人的肩膀
- [【Mybatis源码分析】02-Mapper映射的解析过程](https://blog.csdn.net/shenchaohao12321/article/details/79841046)
- 《玩转 MyBatis：深度解析与定制 》第十章
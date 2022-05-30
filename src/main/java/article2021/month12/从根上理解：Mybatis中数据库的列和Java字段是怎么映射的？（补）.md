## 1. 写在前头
大家好，我是`方圆`，这篇帖子是继[从根上理解：Mybatis中数据库的列和Java字段是怎么映射的](https://github.com/FangYuan33/Java-Enthusiasm/blob/main/src/main/java/article2021/month12/从根上理解：Mybatis中数据库的列和Java字段是怎么映射的？.md) 之后的补充，上一篇帖子我们只在源码层说明了ResultMap是如何被解析的，而没有从具体SQL执行层面上分析`查询出的结果集是如何封装的`，所以这篇帖子就来解决这个问题，两篇合起来，才是它的完整版，废话不多说，开始了

---

## 2. 准备工作

- 需要用到的实体类，非常的简单，只有`id, name, tel`三个字段

```java
public class Dept {

    private Integer id;

    private String name;

    private String tel;
}
```
- 对应的xml文件，定义`简单的select语句`和`ResultMap`

```xml
    <resultMap id="deptMap" type="Dept">
        <id property="id" column="id"/>
        <id property="name" column="name"/>
        <id property="tel" column="tel"/>
    </resultMap>

    <select id="findAll" resultMap="deptMap">
        select * from dept
    </select>
```

- 之后我们Debug的测试用例都是采用`findAll方法`

---
## 3. 从SimpleExecutor的doQuery方法开始
- 执行查询动作的代码在这里，由`StatementHandler`的`query方法`执行查询

```java
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      // StatementHandler执行查询
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

- 我们进入`它的query方法`看看

```java
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 请求数据库查询
    ps.execute();
    // 这个是需要我们重点看的方法，这个方法会处理返回的结果集
    return resultSetHandler.handleResultSets(ps);
  }
```

### 3.1 handleResultSets方法
> `注`：为了可读性，我会把`存储过程`相关的源码全部去掉，因为这个特性实在是用得少，加上又影响文章的流畅，所以大家发现`...`的时候就说明这个地方我略过了无关的代码

- 我们把这个方法拉出来看看，关键的代码行我都标注了章节号，对应着看

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // 3.1.1 将ResultSet封装成ResultSetWrapper
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // 3.1.2 获取我们定义的ResultMap
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      // 3.1.3 处理返回的resultSet结果集
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
    
    // 存储过程相关...

    // 3.1.4 返回我们需要的查询结果
    return collapseSingleResultList(multipleResults);
  }
```

- 主方法中的源码看起来还是很清晰

1. `首先会对ResultSet进行装饰`
2. `之后获取我们定义的ResultMap并处理结果集`
3. `最后会将我们想要的查询结果返回`

- 下面看源码的过程中，我会把执行findAll方法的`Debug过程`也加入进来，我觉得这样有具体的例子更好理解

---

#### 3.1.1 getFirstResultSet方法，将ResultSet封装成ResultSetWrapper

- 还是一样，我们也把getFirstResultSet方法拿出来看看

```java
  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
    ResultSet rs = stmt.getResultSet();
    
    // 存储过程相关...
    
    // 可以发现，只是在这里调用了ResultSetWrapper的构造方法
    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }
```

- 还是很简单，没有复杂的操作，ResultSet之后，调用了`ResultSetWrapper的构造方法`，我们直接看源码

```java
  // 会将元数据列名、java类名和jdbc类型初始化在list中，并且一一对应
  private final List<String> columnNames = new ArrayList<>();
  private final List<String> classNames = new ArrayList<>();
  private final List<JdbcType> jdbcTypes = new ArrayList<>();

  public ResultSetWrapper(ResultSet rs, Configuration configuration) throws SQLException {
    super();
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.resultSet = rs;
    // 这里会对结果集的元数据进行初始化
    final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
      columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
      jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
      classNames.add(metaData.getColumnClassName(i));
    }
  }
```

- 这里的操作非常简单，我把debug截图放出来就非常容易懂了
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/99bbf3f1d1884bd8b30b6da546f68574.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 因为`list插入是有序的`，所以`列的信息会一一对应`，那么在这里初始化这些信息之后就可以拿来直接用，`用空间换时间`的模式


---

#### 3.1.2 获取我们定义的ResultMap
- 这个步骤非常的简单，我就如直接上图了

![在这里插入图片描述](https://img-blog.csdnimg.cn/4d64973a24bf40a9b68d785313c22609.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)

- 我们可以很容易的发现图上红框内的部分都是我们在xml文件中指定的resultMap中的东西

```xml
    <resultMap id="deptMap" type="Dept">
        <id property="id" column="id"/>
        <id property="name" column="name"/>
        <id property="tel" column="tel"/>
    </resultMap>
```

---
#### 3.1.3 handleResultSet方法，处理返回的resultSet结果集
- 这里会对resultSet进行处理，我们直接看else部分，这里会

```java
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      // 存储过程相关，略过
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        // 无resultHandler则走这里
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          // 处理数据的方法，这个我们要接着重点看
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          // 处理完之后，将查询结果对象放入list中
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          // 少人有走的路
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      closeResultSet(rsw.getResultSet());
    }
  }
```

- 多数情况下`resultHandler都是null`，我们进行debug的样例也是一样，都会执行的是使用`默认defaultResultHandler`的代码（附图为执行selectList时resultHandler这个参数就是null）
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/62559421db424a80adac1b34eb62bf57.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 接下来我们还是要继续看`handleRowValues方法`，它的具体执行部分

##### 3.1.3.1 handleRowValues方法
- 这里会`判断resultMap里是否还包含resultMap`，根据结果走不同的逻辑，我们定义的resultMap比较简单，所以走的是下面没有嵌套的逻辑

```java
  public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    // 是否有嵌套的resultMap，有则执行这里
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      // 没有嵌套则执行这里
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
```

- 我们接着看`handleRowValuesForSimpleResultMap方法`

##### 3.1.3.2 handleRowValuesForSimpleResultMap方法

- 下边出现了resultSet，这也就意味着真正的，注释标记好了步骤，其中我们需要重点关注标注`*号`的两行代码，`获取行数据并封装为对象`，和之后的`保存动作`

```java
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    // 这里终于出现了resultSet
    ResultSet resultSet = rsw.getResultSet();
    // 内存分页相关，略过
    skipRows(resultSet, rowBounds);
    // 循环封装返回结果
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      // 鉴别器相关，鉴别器的作用是判断使用哪个resultMap，大家可以去了解一下
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      // *获取行数据并封装为对象
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      // *封装好的对象保存到集合中
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }
```

##### 3.1.3.3 getRowValue方法，获取行数据

- 这个方法调用层级还蛮多的，但是非常简单，我们就在这一小节内都把它说完，标注`*号`的要重点关注

```java
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // *根据结果集类型创建一个空对象（空的实体类）
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      // 创建元对象，之后会用反射来来为属性赋值
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      // *处理自动映射，解释了在我们没有指定resultMap时，依然能完成属性值和列值映射的原因
      // (比如指定resultType的时候)
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      // *处理resultMap中配置好的映射
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      // 如果整个过程没有映射到任何属性，则根据配置决定返回空对象还是null
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }
```

- 创建空对象的`createResultObject方法`，我只暴露出了核心代码，其他复杂的杂七杂八省略掉了，结合注释阅读

```java
  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    ...
    
    // 核心逻辑
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    
    ...
    
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty();
    return resultObject;
  }

  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();

    // 根据对象类型，有以下四种情况
    // 1. 定义了typeHandler，此处处理
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    // 2. 在resultMap中定义了<constructor>子标签
    } else if (!constructorMappings.isEmpty()) {
      return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    // 3. 有默认的无参构造器，则直接借助ObjectFactory创建
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      return objectFactory.create(resultType);
    // 4. 没有默认构造器，则MyBatis会自己试探性寻找合适的构造方法创建对象
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
      return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs);
    }
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
  }
```
多数情况下走的都是第3种，无参构造方法构造对象，我们Debug的例子走的就是这种

- 处理自动映射，执行`applyAutomaticMappings方法`

> 这里我插播一条插曲：在Configuration配置中，有指定自动映射的方式，默认为`AutoMappingBehavior.PARTIAL`，这种方式会开启自动映射但是不能处理包含嵌套的映射

我们看看它的逻辑，分三个步骤：`解析出自动映射`，`获取值`，`反射set值`，如下
```java
  private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // 1. 解析出自动映射
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        // 2. 利用typehandler处理并获取到值
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // 3. 利用反射set属性值
          metaObject.setValue(mapping.property, value);
        }
      }
    }
    return foundValues;
  }
```

- 处理resultMap中配置好的列的映射，`applyPropertyMappings方法`，这个方法有点儿长，但是别担心，非常简单，看着注释一步步来就好

```java
  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    // 获取到我们在resultMap指定的列名们
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;

    // 获取我们指定的属性和列映射关系
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      // 如果指定了列前缀的话，拼接上
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        column = null;
      }
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {

        // 1. 在resultSet中取值
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // 2. 取出对应的属性（还记得上面我们截图的那三个list吗？这里用了）
        final String property = propertyMapping.getProperty();
        if (property == null) {
          continue;
        } else if (value == DEFERRED) {
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // 3. 反射set值
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }
```
这个方法的核心步骤和上边我们分析的自动映射相似，`在resultSet中取值`，`取出对应的属性`，`反射set值`，之后就将我们需要的数据封装在对象里了，之后就是将这个对象放入集合，我们接着往下看


##### 3.1.3.4 封装好的对象保存到集合中，storeObject方法
- 具体看源码中的注释即可

```java
private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, 
        Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
        // 存储过程相关，略过
        linkToParents(rs, parentMapping, rowValue);
    } else {
        // 重点关注这个
        callResultHandler(resultHandler, resultContext, rowValue);
    }
}

// 这个方法分为两步，我们在下面看
private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    // 1.对象存入resultContext
    resultContext.nextResultObject(rowValue);
    // 2. 将对象存入list
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
}
  // 对应1.
  public void nextResultObject(T resultObject) {
    // 内存分页相关，略过
    resultCount++;
    // 这里是直接把封装好的对象存入了resultContext
    this.resultObject = resultObject;
  }

  // 对应2.
  private final List<Object> list;

  public void handleResult(ResultContext<?> context) {
    // 这不是，直接把刚才放进去的对象拿出来存入了list中
    list.add(context.getResultObject());
  }
```

##### 3.1.3.5 不想忽视的步骤：multipleResults.add过程
- 大家回到3.1.3节看一下，这个步骤是在处理完查询结果对象之后将结果加入到`multipleResults`列表里，而这个列表的字面意思是`多个结果`，这是针对存储过程而言的，我们没有使用存储过程，所以返回的结果只有一个，出现`multipleResults列表中保存有一个结果列表`的形式，如下图所示

![在这里插入图片描述](https://img-blog.csdnimg.cn/1073fb5737004ddb9e896955a7584481.png#pic_center)

- 接着往下看能更好的理解这一点

---
#### 3.1.4 collapseSingleResultList方法，返回我们需要的查询结果
- 好了，这不就到了最后一步了嘛。注意这里判断了一下`multipleResults的大小`，结合前边我们刚说的，没使用存储过程，列表里只有一个对象，拿出来返回，如果是存储过程，就直接返回的是multipleResults

```java
private List<Object> collapseSingleResultList(List<Object> multipleResults) {
    return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
}
```

---

#### 3.1.5 番外 - 指定resultType时，映射的实现
- 上文中我们有提到`applyAutomaticMappings自动映射方法`，我们debug看一下，在我们没有指定resultMap时，它是如何实现的列与属性的映射。我们在xml文件中指定好resultType

```xml
    <select id="findAll" resultType="Dept">
        select * from dept
    </select>
```
- 我们debug直接进来，发现autoMapping生成了我们没指定的`列与属性的映射`
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/a0b9bd48e6ec48e9ab8969c320e7bdcf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 那么我们进入`createAutomaticMappings方法`看看，是怎么生成的，大家重点关注注释即可，别嫌代码长，看注释！

```java
  private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // 注意这个resultMap'平平无奇'，它不包含任何映射关系，见下方图片
    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    // 这里get的映射列表为null
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    
    if (autoMapping == null) {
      autoMapping = new ArrayList<>();
      // 获取到未映射的列名（id, name, tel)
      final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
      for (String columnName : unmappedColumnNames) {
        // 重点：这里直接让属性名为列名
        String propertyName = columnName;

        // 拼接前缀
        if (columnPrefix != null && !columnPrefix.isEmpty()) {
          if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
            propertyName = columnName.substring(columnPrefix.length());
          } else {
            continue;
          }
        }
        // 根据属性名找到对应的属性，并将列与属性的映射关系封装到autoMapping中
        final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
        if (property != null && metaObject.hasSetter(property)) {
          if (resultMap.getMappedProperties().contains(property)) {
            continue;
          }
          final Class<?> propertyType = metaObject.getSetterType(property);
          if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
            final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
            autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
          } else {
            configuration.getAutoMappingUnknownColumnBehavior()
                .doAction(mappedStatement, columnName, property, propertyType);
          }
        } else {
          configuration.getAutoMappingUnknownColumnBehavior()
              .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
        }
      }
      autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
  }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/5090a667f93947c5b3314784d4364a37.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5pa55ZyG5oOz5b2T5Zu-54G1,size_20,color_FFFFFF,t_70,g_se,x_16)
- 我们可以发现mybatis`默认把属性名指定为了列名`，再根据这个属性名再去匹配属性，成功匹配的生成autoMapping映射

- 我们再回到applyAutomaticMappings方法中，再捋一下这个流程

```java
  private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // 根据列名生成了自动映射
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        // 获取值
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // 反射set值
          metaObject.setValue(mapping.property, value);
        }
      }
    }
    return foundValues;
  }
```

- 还是之前我们提到的老步骤，`解析出自动映射`，`获取值`，`反射set值`。

---

## 4. Over
终于写完了，来回来去的将近写了一小天，也没那么难，如果你真的看完了，文中有不对和你觉得能完善的地方，非常欢迎提出来，好了，拜拜
* **入参枚举说明**

``` 
EnumOrdinalTypeHandler 
```

```
EnumTypeHandler
```
> 在入参使用枚举时,parameterType没有指定枚举类型,需要在```#{menu,javaType=com.fanyin...,typeHanlder=org.apache.ibatis.type.EnumOrdinalTypeHandler}```中指定否则注册枚举处理类型时,枚举类型为空会抛异常

* **自定义枚举**

> 继承 ```BaseTypeHandler```类,如果需要处理的枚举类型一致,只需要使用`@MappedTypes`传入多个需要解析的枚举类即可

* **自定义ResultHandler** 

> 实现该接口,可以自定义返回的数据结果集

```java
ResultHandler handler = new MyResultHandler();
public interface UserMapper {
  void getMapList(ResultHandler handler);
}
```



## 源码分析

####  MapperScannerRegistrar

> `@MapperScan` 注解会导入该类,该类主要用于注册BeanDefinitions,怎么获取`BeanDefinition`数据呢? 如下

#### ClassPathMapperScanner

> 该类继承`ClassPathBeanDefinitionScanner` 类,因此具备根据包名扫描类并生成`BeanDefinition`的功能
>
> 但即便是生成接口的`BeanDefinition`依旧无法实例化,因此该类会在扫描后的`BeanDefinition`后,重新加工一下,使其变为一个`MapperFactoryBean`的FactoryBean,而在其getObject()中定义真实的"接口实现类"

#### MybatisAutoConfiguration

> 该类为自动化配置的类,会生成`SqlSessionFactory` 和 `SqlSessionTemplate` 两个Bean
>
> 如果不是使用`@MapperScan` 而是使用的`@Mapper`,则会额外注册`AutoConfiguredMapperScannerRegistrar`,用于处理包含`@Mapper`接口的类
>
> `SqlSessionFactory` 是由`SqlSessionFactoryBean.getObject()`进行创建,步骤如下:

* 设置`Configuration`配置文件中约定的属性
* 设置`Configuration` 环境(`DataSource`,事务)
* 解析`mybatis.mapper-locations` 中定义的xml并将解析结果保存在`Configuration`中

```java
protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
protected final InterceptorChain interceptorChain = new InterceptorChain();
protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
//以Mapper.xml中标签id为key缓存,insert update delete select标签解析对象
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
//cache标签解析 很少使用
protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");
//resultMap标签解析
protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
//parameterMap标签解析,很少使用
protected final Map<String, ParameterMap> parameterMaps = new StrictMap<ParameterMap>("Parameter Maps collection");
//insert中子标签selectKey标签解析,用于插入后返回自增的id
protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<KeyGenerator>("Key Generators collection");

```



#### MapperFactoryBean

> 主要生成一个基于`MapperProxy`的动态对象(基于java原生的动态代理)
>
> 该类中通过set方法自动注入`SqlSessionFactory`和`SqlSessionTemplate` 
>
> 且在注入时会创建<a href="#sqlSessionTemplate">SqlSessionTemplate</a>对象

```java
//DefaultSqlSession
public <T> T getMapper(Class<T> type) {
   return configuration.<T>getMapper(type, this);
}
//MapperRegistry
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  //该处会以接口类为key做了一个缓存
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}

//MapperProxyFactory
//以方法为基本单位进行缓存
private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

#### MapperProxy

> 一个Mapper(业务中定义的查询接口)代理的对象

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    //object类中相关方法
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else if (isDefaultMethod(method)) {
      //接口中定义的默认方法,例如 1.8中default方法和静态方法
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  //执行查询数据库等操作
  return mapperMethod.execute(sqlSession, args);
}
```

#### MapperMethod

```java
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
  //用于判断Sql的类型 insert update delete select flush等
  this.command = new SqlCommand(config, mapperInterface, method);
  //方法入参和出参类型分析等	@MapKey:如果返回Map<Long,User>类型时,key以数据哪个字段为准
  this.method = new MethodSignature(config, mapperInterface, method);
}

public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  // update insert delete三个方法底层实现实际上是同一个方法
  switch (command.getType()) {
    case INSERT: {
      //将多个参数变成Map,因为底层是通过Map作为参数进行传递的
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      //如果Mapper指定ResultHandler,则按用户自定义的方式进行填充,用于返回特殊结构的数据类型
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        //返回list
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        //返回map
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        //应该和存储过程相关,没用过
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        //单个
        result = sqlSession.selectOne(command.getName(), param);
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
                               + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
//底层 增删改
public int update(String statement, Object parameter) {
  try {
    dirty = true;
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.update(ms, wrapCollection(parameter));
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}


```

#### <a id="sqlSessionTemplate">SqlSessionTemplate</a>

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                          PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
  notNull(executorType, "Property 'executorType' is required");

  this.sqlSessionFactory = sqlSessionFactory;
  this.executorType = executorType;
  this.exceptionTranslator = exceptionTranslator;
  //通过jdk动态代理生成sqlSession对象
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
    SqlSessionFactory.class.getClassLoader(),
    new Class[] { SqlSession.class },
    new SqlSessionInterceptor());
}
private class SqlSessionInterceptor implements InvocationHandler {
  
    //在调用selectOne,selectList,insert等方法会执行该方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      //创建的SqlSession=DefaultSqlSession,因此最后执行selectOne等方法时,还是会调用DefaultSqlSession相应的方法
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        //不是spring管理事务
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          //不在spring事务管理时,才会关闭sqlSession,否则由spring关闭
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

#### DefaultSqlSession

```java
//SimpleExecutor,Sqlsession.update会调用Executor#update方法
@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    //sql处理器
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    //创建PrepareStatement,参数填充等
    stmt = prepareStatement(handler, ms.getStatementLog());
    //真实执行update方法
    return handler.update(stmt);
  } finally {
    closeStatement(stmt);
  }
}
//Configuration
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  //走一波拦截器,可以在拦截器里进行自定义扩展
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
//创建的RoutingStatementHandler也是实现StatementHandler接口,且最终会委派给PreparedStatementHandler处理
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


private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  //还是基于动态代理的方式创建(用于记录日志)
  Connection connection = getConnection(statementLog);
  //初始化PrepareStatement,会调用instantiateStatement方法
  stmt = handler.prepare(connection, transaction.getTimeout())
  //?占位符设置参数
  handler.parameterize(stmt);
  return stmt;
}

protected Statement instantiateStatement(Connection connection) throws SQLException {
  String sql = boundSql.getSql();
  if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
    String[] keyColumnNames = mappedStatement.getKeyColumns();
    if (keyColumnNames == null) {
      return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
    } else {
      return connection.prepareStatement(sql, keyColumnNames);
    }
  } else if (mappedStatement.getResultSetType() != null) {
    return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
  } else {
    return connection.prepareStatement(sql);
  }
}

//DefaultParameterHandler 
public void setParameters(PreparedStatement ps) {
  ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
  //ParameterMapping是解析xml中#{xx}属性后生成的一个描述信息类
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        // #{id,jdbcType=INTEGER} id即为property
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
          //基本类型可以直接赋值
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          //parameterObject 可能为一个对象,也可能为Map,该处通过反射获取属性或value值
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        //typeHandler可以在配置文件中指定 #{id,jdbcType=INTEGER,typeHandler=com.eghm.MyTypeHandler	}
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
        JdbcType jdbcType = parameterMapping.getJdbcType();
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
        try {
          //设置值
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        } catch (TypeException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        } catch (SQLException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        }
      }
    }
  }
}

// SimpleExecutor 查询单个和查询list都是同一个方法 ,与doUpdate方法有区别的就是,要映射结果集,分页信息等 
public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
  throws SQLException {
  Statement stmt = null;
  try {
    flushStatements();
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
    Connection connection = getConnection(ms.getStatementLog());
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return handler.<E>query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

#### DefaultResultSetHandler

> 用于处理结果集到java类之间的映射的

```java
@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
	
  final List<Object> multipleResults = new ArrayList<Object>();

  int resultSetCount = 0;
  //将ResultSet进行包装,并解析查询结果的字段名,属性等信息
  ResultSetWrapper rsw = getFirstResultSet(stmt);
	// xml中的resultMap标签代表一个ResultMap对象,一个<column>标签代表一个ResultMapping对象
  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
  validateResultMapsCount(rsw, resultMapCount);
  while (rsw != null && resultMapCount > resultSetCount) {
    ResultMap resultMap = resultMaps.get(resultSetCount);
    //解析数据并将数据填充到multipleResults中
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }

  String[] resultSets = mappedStatement.getResultSets();
  if (resultSets != null) {
    while (rsw != null && resultSetCount < resultSets.length) {
      ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
      if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
      }
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
  }
  return collapseSingleResultList(multipleResults);
}

//数据填充
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
  try {
    if (parentMapping != null) {
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
    } else {
      //没有手动指定ResultHandler,由系统自动处理
      if (resultHandler == null) {
        DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
        multipleResults.add(defaultResultHandler.getResultList());
      } else {
        //交由用户来处理相关逻辑
        handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
      }
    }
  } finally {
    // issue #228 (close resultsets)
    closeResultSet(rsw.getResultSet());
  }
}

public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  //表示<resultMap>标签中内嵌<association>,<collection>等子标签
  if (resultMap.hasNestedResultMaps()) {
    ensureNoRowBounds();
    checkResultHandler();
    handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  } else {
    handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  }
}
//没有内嵌子标签
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
  throws SQLException {
  DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
  //内存分页时使用,一般不用
  skipRows(rsw.getResultSet(), rowBounds);
  //遍历ResultSet的每一行数据
  while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
    //对resultMap标签中定义了discriminator子标签做额外映射处理, discriminator类似于switch,对查询的结果根据条件进行额外映射或者其他处理
    ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
    Object rowValue = getRowValue(rsw, discriminatedResultMap);
    storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
  }
}

private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
  final ResultLoaderMap lazyLoader = new ResultLoaderMap();
  //根据返回值类型创建相应的对象,如果需要延迟加载,则会创建基于返回值的代理对象
  Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);
  //没有发现与返回值类型匹配的TypeHandler,交由系统自动映射
  if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final MetaObject metaObject = configuration.newMetaObject(rowValue);
    boolean foundValues = this.useConstructorMappings;
    if (shouldApplyAutomaticMappings(resultMap, false)) {
      //根据查询结果字段进行反向判断,如果开启autoMapping=true,即便是resultMap没有定义的字段,只要相应实例类中有与查询结果列一样的属性,依旧会自动映射
      //如果查询结果通过resultType=com.eghm...指定,依旧会动态与类绑定
      foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
    }
    //<column>标签 赋值
    foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
    foundValues = lazyLoader.size() > 0 || foundValues;
    rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
  }
  return rowValue;
}

// resultMap标签中包含嵌套的子标签
private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  final DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
  skipRows(rsw.getResultSet(), rowBounds);
  Object rowValue = previousRowValue;
  while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
    final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
    final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
    Object partialObject = nestedResultObjects.get(rowKey);
    // issue #577 && #542
    if (mappedStatement.isResultOrdered()) {
      if (partialObject == null && rowValue != null) {
        nestedResultObjects.clear();
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
      }
      rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
    } else {
      rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      if (partialObject == null) {
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
      }
    }
  }
  if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
    storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    previousRowValue = null;
  } else if (rowValue != null) {
    previousRowValue = rowValue;
  }
}

```



#### TypeHandler

> 用于查询条件映射和结果集字段映射(即数据库出入参类型与java类型之间的映射关系)
>
> 在自定义的`TypeHandler`中添加`MappedTypes` 注解,这样在请求或者响应中,如果对象为注解中指定的类型,自动会通过自定义的TypeHandler来处理

#### XMLMapperBuilder

> 解析整个xml文件的类

#### ResultMapResolver

> 解析 resultMap标签

#### XMLStatementBuilder

> 解析 insert update delete select标签

#### XMLScriptBuilder

> 解析insert update delete select中的子标签 例如 if set include foreach

#### MapperBuilderAssistant

> 将解析的标签属性转换相应的类对象

#### SqlNode

> 用来解析动态sql,即if,trim,bind,where,set,choose,otherwise,when,foreach

#### SqlSourceBuilder

> 解析#{},并将其替换为?
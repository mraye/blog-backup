---
title: spring结合mybatis时一级缓存失效问题
date: 2018-05-26 21:37:09
tags: [mybatis, cache]
categories: [mybatis]
---



> 之前了解到mybatis的一级缓存是默认开启的，作用域是sqlSession,是基 HashMap的本地缓存。不同的SqlSession之间的缓存数据区域互不影响。当进行select、update、delete操作后并且commit事物到数据库之后，sqlSession中的Cache自动被清空

```xml
<setting name="localCacheScope" value="SESSION"/>
```

#### 结论

spring结合mybatis后，一级缓存作用：  

+ 在未开启事物的情况之下，每次查询，spring都会关闭旧的sqlSession而创建新的sqlSession,因此此时的一级缓存是没有启作用的
+ 在开启事物的情况之下，spring使用threadLocal获取当前资源绑定同一个sqlSession，因此此时一级缓存是有效的

<!-- more -->

#### 案例

情景一:未开启事物

```java

@Service("countryService")
public class CountryService {

    @Autowired
    private CountryDao countryDao;

    // @Transactional 未开启事物
    public void noTranSactionMethod() throws JsonProcessingException {
        CountryDo countryDo = countryDao.getById(1L);
        CountryDo countryDo1 = countryDao.getById(1L);
        ObjectMapper objectMapper = new ObjectMapper();
        String json = objectMapper.writeValueAsString(countryDo);
        String json1 = objectMapper.writeValueAsString(countryDo1);
        System.out.println(json);
        System.out.println(json1);
    }
}
```

测试案例:  

```java
@Test
public void transactionTest() throws JsonProcessingException {
    countryService.noTranSactionMethod();
}
```

结果:  

```console
[DEBUG] SqlSessionUtils Creating a new SqlSession
[DEBUG] SpringManagedTransaction JDBC Connection [com.mysql.jdbc.JDBC4Connection@14a54ef6] will not be managed by Spring
[DEBUG] getById ==>  Preparing: SELECT * FROM country WHERE country_id = ?
[DEBUG] getById ==> Parameters: 1(Long)
[DEBUG] getById <==      Total: 1
[DEBUG] SqlSessionUtils Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@3359c978]
[DEBUG] SqlSessionUtils Creating a new SqlSession
[DEBUG] SqlSessionUtils SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@2aa27288] was not registered for synchronization because synchronization is not active
[DEBUG] SpringManagedTransaction JDBC Connection [com.mysql.jdbc.JDBC4Connection@14a54ef6] will not be managed by Spring
[DEBUG] getById ==>  Preparing: SELECT * FROM country WHERE country_id = ?
[DEBUG] getById ==> Parameters: 1(Long)
[DEBUG] getById <==      Total: 1
[DEBUG] SqlSessionUtils Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@2aa27288]
{"countryId":1,"country":"Afghanistan","lastUpdate":"2006-02-15 04:44:00.0"}
{"countryId":1,"country":"Afghanistan","lastUpdate":"2006-02-15 04:44:00.0"}
```

**可以看到，两次查询，都创建了新的sqlSession,并向数据库查询，此时缓存并没有起效果**


情景二: 开启事物

打开`@Transactional`注解:  

```java
@Service("countryService")
public class CountryService {

    @Autowired
    private CountryDao countryDao;

    @Transactional
    public void noTranSactionMethod() throws JsonProcessingException {
        CountryDo countryDo = countryDao.getById(1L);
        CountryDo countryDo1 = countryDao.getById(1L);
        ObjectMapper objectMapper = new ObjectMapper();
        String json = objectMapper.writeValueAsString(countryDo);
        String json1 = objectMapper.writeValueAsString(countryDo1);
        System.out.println(json);
        System.out.println(json1);
    }
}
```

使用原来的测试案例，输出结果:  

```console
[DEBUG] SqlSessionUtils Creating a new SqlSession
[DEBUG] SqlSessionUtils Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@109f5dd8]
[DEBUG] SpringManagedTransaction JDBC Connection [com.mysql.jdbc.JDBC4Connection@55caeb35] will be managed by Spring
[DEBUG] getById ==>  Preparing: SELECT * FROM country WHERE country_id = ?
[DEBUG] getById ==> Parameters: 1(Long)
[DEBUG] getById <==      Total: 1
[DEBUG] SqlSessionUtils Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@109f5dd8]
// 从当前事物中获取sqlSession
[DEBUG] SqlSessionUtils Fetched SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@109f5dd8] from current transaction
[DEBUG] SqlSessionUtils Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@109f5dd8]
{"countryId":1,"country":"Afghanistan","lastUpdate":"2006-02-15 04:44:00.0"}
{"countryId":1,"country":"Afghanistan","lastUpdate":"2006-02-15 04:44:00.0"}
```

**可以看到，两次查询，只创建了一次sqlSession,说明一级缓存起作用了**


#### 跟踪源码

从`SqlSessionDaoSupport`作为路口，这个类在`mybatis-spring`包下，sping为sqlSession做了代理

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {

  private SqlSession sqlSession;

  private boolean externalSqlSession;

  public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
  //....omit
}
```

创建了`SqlSessionTemplate`后,在`SqlSessionTemplate`中:  

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
  notNull(executorType, "Property 'executorType' is required");

  this.sqlSessionFactory = sqlSessionFactory;
  this.executorType = executorType;
  this.exceptionTranslator = exceptionTranslator;
  //代理了SqlSession
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
      SqlSessionFactory.class.getClassLoader(),
      new Class[] { SqlSession.class },
      new SqlSessionInterceptor());
}
```

再看`SqlSessionInterceptor`,`SqlSessionInterceptor`是`SqlSessionTemplate`的内部类:  

```java

public class SqlSessionTemplate implements SqlSession, DisposableBean {
  // ...omit..
    private class SqlSessionInterceptor implements InvocationHandler {
     @Override
     public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
       SqlSession sqlSession = getSqlSession(
           SqlSessionTemplate.this.sqlSessionFactory,
           SqlSessionTemplate.this.executorType,
           SqlSessionTemplate.this.exceptionTranslator);
       try {
         Object result = method.invoke(sqlSession, args);
         //如果尚未开启事物(事物不是由spring来管理)，则sqlSession直接提交
         if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
           // force commit even on non-dirty sessions because some databases require
           // a commit/rollback before calling close()
           // 手动commit
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
         //一般情况下，默认都是关闭sqlSession
         if (sqlSession != null) {
           closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
         }
       }
     }
    }
}
```


再看`getSqlSession`方法，这个方法是在`SqlSessionUtils.java`中的：  

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
  notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
  //获取holder
  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
  //从sessionHolder中获取SqlSession
  SqlSession session = sessionHolder(executorType, holder);
  if (session != null) {
    return session;
  }

  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug("Creating a new SqlSession");
  }

  //如果sqlSession不存在，则创建一个新的
  session = sessionFactory.openSession(executorType);
  //将sqlSession注册在sessionHolder中
  registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

  return session;
}


private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
    SqlSessionHolder holder;
    //在开启事物的情况下
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      Environment environment = sessionFactory.getConfiguration().getEnvironment();

      //由spring来管理事物的情况下
      if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registering transaction synchronization for SqlSession [" + session + "]");
        }

        holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
        //将sessionFactory绑定在sessionHolde相互绑定
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);
        TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
        holder.setSynchronizedWithTransaction(true);
        holder.requested();
      } else {
        if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
          if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
          }
        } else {
          throw new TransientDataAccessResourceException(
              "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
      }
    }

```


再看`TransactionSynchronizationManager.bindResource`的方法:  


```java

public abstract class TransactionSynchronizationManager {

    //omit...
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");

      // key:sessionFactory, value:SqlSessionHolder(Connection)
      public static void bindResource(Object key, Object value) throws IllegalStateException {
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        Assert.notNull(value, "Value must not be null");
        //从threadLocal类型的resources中获取与当前线程绑定的资源，如sessionFactory，Connection等等
        Map<Object, Object> map = resources.get();
        // set ThreadLocal Map if none found
        if (map == null) {
          map = new HashMap<Object, Object>();
          resources.set(map);
        }
        Object oldValue = map.put(actualKey, value);
        // Transparently suppress a ResourceHolder that was marked as void...
        if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
          oldValue = null;
        }
        if (oldValue != null) {
          throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
              actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
        }
        if (logger.isTraceEnabled()) {
          logger.trace("Bound value [" + value + "] for key [" + actualKey + "] to thread [" +
              Thread.currentThread().getName() + "]");
        }
      }
}
```

**这里可以看到，spring是如何做到获取到的是同一个SqlSession,前面的长篇大论，就是为使用ThreadLocal将当前线程绑定创建SqlSession相关的资源，从而获取同一个sqlSession**

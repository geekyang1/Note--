# Godbatis思路

- 因为才开始写肯定不知道咋写，借助配置文件，从而分析思路
- 在原先写mybatis代码时我们知道要使用mybatis需要使用三个对象 sqlSessionFactoryBuild sqlSessionFactory sqlSession
- 所以我们要创建这三个对象 
- 由三个对象关系来看第一步：sqlSessionFactoryBuild  第二步：sqlSessionFactory  第三步：sqlSession
- sqlSessionFactoryBuild在这个对象中提供build 方法需要返回sqlSessisonFactory对象，并且穿一个流，这个流帮助sqlSessisonFactory去寻找配置文件，从而解析配置文件
- ![image-20230411094458013](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20230411094458013.png)

创建sqlSessisonFactory对象之后分析他的作用就是：解析godbatis—config.xml文件。

然后分析该文件标签中的作用：

1. ## 需要事务管理器（其中有JDBC，MANAGE两种不同的选择）

   创建一个接口，并且当作sqlSessionFactory中的一个属性，面向接口编程，以多态的方式注入到接口中

   然后在创建JDBC，MANAGE两个不同的实现类，去实现接口中的方法

   但是在事务中需要有连接对象Connection，所以就需要数据源来提供

   ```java
   package org.god.core;
   
   import java.sql.Connection;
   
   /**
    * 事务管理器接口
    * @author 老杜
    * @version 1.0
    * @since 1.0
    */
   public interface TransactionManager {
       /**
        * 提交事务
        */
       void commit();
   
       /**
        * 回滚事务
        */
       void rollback();
   
       /**
        * 关闭事务
        */
       void close();
   
       /**
        * 开启连接
        */
       void openConnection();
       
       /**
        * 获取连接对象
        * @return 连接对象
        */
       Connection getConnection();
   }
   
   ```

   数据源获取完毕之后，需要在每个方法中用Connection对象执行相应方法，但是他在一个事务中需要同一个Connection对象才能保证在同一个事务当中。

   才用的设计方法是提供一个私有化Connection对象，并且提供一个openConnection方法，如果判断Connection为null就给Connection赋值，如果不为空则不执行

   ```java
   	/**
        * 连接对象，控制事务时需要
        */
       private Connection conn;
    	@Override
       public void openConnection() {
           if（conn==null）{
               conn=dataSource.getConnection
           }
       }
   ```

   为了能让sql语句执行时使用同一个Connection对象需要提供一个构造方法getConnection（）；

   ```java
   @Override
       public Connection getConnection() {
           return conn;
       }
   ```

   下面是完整版的JDBCTranscation实现类

   ```java
   package org.god.core;
   
   import javax.sql.DataSource;
   import java.sql.Connection;
   import java.sql.SQLException;
   
   /**
    * 事务管理器
    * @author 老杜
    * @version 1.0
    * @since 1.0
    */
   public class GodJDBCTransaction implements TransactionManager {
       /**
        * 连接对象，控制事务时需要
        */
       private Connection conn;
   
       /**
        * 数据源对象
        */
       private DataSource dataSource;
   
       /**
        * 自动提交标志：
        * true表示自动提交
        * false表示不自动提交
        */
       private boolean autoCommit;
   
       /**
        * 构造事务管理器对象
        * @param autoCommit
        */
       public GodJDBCTransaction(DataSource dataSource, boolean autoCommit) {
           this.dataSource = dataSource;
           this.autoCommit = autoCommit;
       }
   
       /**
        * 提交事务
        */
       public void commit(){
           try {
               conn.commit();
           } catch (SQLException e) {
               throw new RuntimeException(e);
           }
       }
   
       /**
        * 回滚事务
        */
       public void rollback(){
           try {
               conn.rollback();
           } catch (SQLException e) {
               throw new RuntimeException(e);
           }
       }
   
       @Override
       public void close() {
           try {
               conn.close();
           } catch (SQLException e) {
               throw new RuntimeException(e);
           }
       }
   
       @Override
       public void openConnection() {
           try {
               this.conn = dataSource.getConnection();
               this.conn.setAutoCommit(this.autoCommit);
           } catch (SQLException e) {
               throw new RuntimeException(e);
           }
       }
   
       @Override
       public Connection getConnection() {
           return conn;
       }
   }
   ```

   

2. ## 需要数据源（获取connection对象 简称为数据源）

   由需要事务管理器分析出需要数据源，所以开始写数据源。

   **由此分析到貌似sqlSessionFactory不用负责数据员，它只在事务管理器中用，可以去掉这一个方法（不知道先写上，写到具体的时候在进行调整）**

   同样数据源的选项也有三个  UNPOOLED（不使用连接池，每使用一次创建一个新的Connection对象） 

   ​												POOLED（使用godbatis自带的连接池）

   ​												JNDI（使用第三方服务器提供的连接池）

   因为有三个选项，所以同样需要面向接口编程，用多态的方式对UNPOOLED，POOLED，JNDI进行注入选择。

   幸运的是JDK已经提供了这么一个公共接口要求所有数据源必须对其实现javax.sql.DataSource(这样我们就可以不用去写接口了)

   同样我们需要在需要事务管理器中提供接口属性方便去实现它。

   # 3.封装sql对象（mapperStatment）

   这一步就相当于从godbatis-config.xml文件跳转到XxxMapper.xml文件中

   封装sql对象其实就是在封装Mapper文件里面的标签

   所以这个对象需要有sql语句标签的内容 resultType, sql 所以需要这两个属性。

   ## sqlSessionFactoryBuild对象中build方法的完善。

   首先使用dom4j解析配置文件，获得一个个标签，在针对于不同的标签去创建对象，具体看如下代码。

   这里面的内容值得研究。

   ```java
   package org.god.core;
   
   import org.dom4j.Document;
   import org.dom4j.DocumentException;
   import org.dom4j.Element;
   import org.dom4j.io.SAXReader;
   
   import javax.sql.DataSource;
   import java.io.InputStream;
   import java.util.HashMap;
   import java.util.Map;
   
   /**
    * SqlSessionFactory对象构建器
    *
    * @author 老杜
    * @version 1.0
    * @since 1.0
    */
   public class SqlSessionFactoryBuilder {
   
       /**
        * 创建构建器对象
        */
       public SqlSessionFactoryBuilder() {
       }
   
   
       /**
        * 获取SqlSessionFactory对象
        * 该方法主要功能是：读取godbatis核心配置文件，并构建SqlSessionFactory对象
        *
        * @param inputStream 指向核心配置文件的输入流
        * @return SqlSessionFactory对象
        */
       public SqlSessionFactory build(InputStream inputStream) throws DocumentException {
           SAXReader saxReader = new SAXReader();
           Document document = saxReader.read(inputStream);
           Element environmentsElt = (Element) document.selectSingleNode("/configuration/environments");
           String defaultEnv = environmentsElt.attributeValue("default");
           Element environmentElt = (Element) document.selectSingleNode("/configuration/environments/environment[@id='" + defaultEnv + "']");
           // 解析配置文件，创建数据源对象
           Element dataSourceElt = environmentElt.element("dataSource");
           DataSource dataSource = getDataSource(dataSourceElt);
           // 解析配置文件，创建事务管理器对象
           Element transactionManagerElt = environmentElt.element("transactionManager");
           TransactionManager transactionManager = getTransactionManager(transactionManagerElt, dataSource);
           // 解析配置文件，获取所有的SQL映射对象
           Element mappers = environmentsElt.element("mappers");
           Map<String, GodMappedStatement> mappedStatements = getMappedStatements(mappers);
           // 将以上信息封装到SqlSessionFactory对象中
           SqlSessionFactory sqlSessionFactory = new SqlSessionFactory(transactionManager, mappedStatements);
           // 返回
           return sqlSessionFactory;
       }
   
       private Map<String, GodMappedStatement> getMappedStatements(Element mappers) {
           Map<String, GodMappedStatement> mappedStatements = new HashMap<>();
           mappers.elements().forEach(mapperElt -> {
               try {
                   String resource = mapperElt.attributeValue("resource");
                   SAXReader saxReader = new SAXReader();
                   Document document = saxReader.read(Resources.getResourcesAsStream(resource));
                   Element mapper = (Element) document.selectSingleNode("/mapper");
                   String namespace = mapper.attributeValue("namespace");
   
                   mapper.elements().forEach(sqlMapper -> {
                       String sqlId = sqlMapper.attributeValue("id");
                       String sql = sqlMapper.getTextTrim();
                       String parameterType = sqlMapper.attributeValue("parameterType");
                       String resultType = sqlMapper.attributeValue("resultType");
                       String sqlType = sqlMapper.getName().toLowerCase();
                       // 封装GodMappedStatement对象
                       GodMappedStatement godMappedStatement = new GodMappedStatement(sqlId, resultType, sql, parameterType, sqlType);
                       mappedStatements.put(namespace + "." + sqlId, godMappedStatement);
                   });
   
               } catch (DocumentException e) {
                   throw new RuntimeException(e);
               }
           });
           return mappedStatements;
       }
   
   
       private TransactionManager getTransactionManager(Element transactionManagerElt, DataSource dataSource) {
           String type = transactionManagerElt.attributeValue("type").toUpperCase();
           TransactionManager transactionManager = null;
           if ("JDBC".equals(type)) {
               // 使用JDBC事务
               transactionManager = new GodJDBCTransaction(dataSource, false);
           } else if ("MANAGED".equals(type)) {
               // 事务管理器是交给JEE容器的
           }
           return transactionManager;
       }
   
       private DataSource getDataSource(Element dataSourceElt) {
           // 获取所有数据源的属性配置
           Map<String, String> dataSourceMap = new HashMap<>();
           dataSourceElt.elements().forEach(propertyElt -> {
               dataSourceMap.put(propertyElt.attributeValue("name"), propertyElt.attributeValue("value"));
           });
   
           String dataSourceType = dataSourceElt.attributeValue("type").toUpperCase();
           DataSource dataSource = null;
           if ("POOLED".equals(dataSourceType)) {
   
           } else if ("UNPOOLED".equals(dataSourceType)) {
               dataSource = new GodUNPOOLEDDataSource(dataSourceMap.get("driver"), dataSourceMap.get("url"), dataSourceMap.get("username"), dataSourceMap.get("password"));
           } else if ("JNDI".equals(dataSourceType)) {
   
           }
           return dataSource;
       }
   }
   
   ```

   


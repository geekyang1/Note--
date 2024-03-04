# Mybatis缓存

## mybatis的缓存只对于DQL语句生效（即select查询语句），DML语句都是现连数据库的。

## 缓存的作业其实就是通过减少IO的方式提高效率

缓存是一种优化技术，提高复用率。

mybatis框架让你在查询同一条记录时不用在去硬盘中取，而是把该sql语句加载到内存中。

这样可以减少io损耗的时间

![](E:\Note\Mybatis\Mybayis缓存\Mybatis缓存.png)

# Mybatis的一二级缓存

- 一级缓存 ：将查询到的数据存储到SqlSession对象中。

  1. 什么时候不走一级缓存

     ​	sqlsession对象不是同一个的时候不走一级缓存

​					查询条件不同也不走一级缓存

​			2.什么时候一级缓存失效

​					手动执行清楚一级缓存的方法 sqlsession.clearCache（）；

​					执行DML语句中的任意一条都会清除一级缓存。		

- 二级缓存：将查询到的数据存储到SqlSessionFactory对象中 

​				如何使用二级缓存：

​													这四个条件必须都具备才能使用二级缓存

​				1.<setting name="cacheEnabled" value="true"> 全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。默认就是true，无需设置。

​				2.使用二级缓存需要在对应的XxxMapper.xml文件中配置<cache>标签

​				3.使用二级缓存的实体类对象必须是可序列化的，即实现java.io.Serializable接口

​				4.Sqlsession对象提交或者关闭以后，一级缓存才会被写入到二级缓存中

- 或者集成其它第三方的缓存：比如EhCache【Java语言开发的】、Memcache【C语言开发的】等。

  了解 详情参考老杜语雀笔记[MyBatis (yuque.com)](https://www.yuque.com/dujubin/ltckqu/pozck9?#eWTHI)

  第十四章最后一节


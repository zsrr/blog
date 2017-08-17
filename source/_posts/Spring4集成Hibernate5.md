---
title: Spring4集成Hibernate5
date: 2017-08-07 18:47:05
tags: [Java Web,Hibernate,Spring]
categories: Spring

---
本篇记录一下Spring与Hibernate的集成，所使用的是Spring4和Hibernate5。
<!--more-->

# 基本配置
## 配置FactoryBean
为了可以在DAO中注入SessionFactory，需要配置Spring提供的便利类**org.springframework.orm.hibernate5.LocalSessionFactoryBean**，注意版本号，不同的Hibernate版本用到的FactoryBean是不一样的：

	@Bean
    LocalSessionFactoryBean factoryBean(DataSource dataSource) {
        LocalSessionFactoryBean sfb = new LocalSessionFactoryBean();
        sfb.setDataSource(dataSource);
        sfb.setAnnotatedPackages("com.stephen.bangbang.domain");
        sfb.setPackagesToScan("com.stephen.bangbang.domain");
        
        // 要配置的属性，对应于hibernate.cfg.xml，也可以使用xml文件配置，但是两者不能同时使用
        Properties properties = new Properties();
        properties.put("hibernate.show_sql", "true");
        properties.put("hibernate.format_sql", "true");
        
        ...
        
        sfb.setHibernateProperties(properties);
        return sfb;
    }
注意可以使用**LocalSessionFactoryBean#setConfigLocation()**方法使用传统的**hibernate.cfg.xml**文件来配置Hibernate，但是据笔者自测，似乎不能和上述展示的手动配置Properties的方法同时使用，使用配置文件也很难进行动态配置，因为**hibernate.cfg.xml**文件似乎不能引入外值。
## 根据Profile动态配置
有时候要根据Profile选择不同的数据源，相应的有不同的Hibernate配置，例如方言(Dialect)和**hibernate.hbm2ddl.auto**属性，关于**hibernate.hbm2ddl.auto**，参见：[hibernate.hbm2ddl.auto配置详解](http://www.cnblogs.com/talo/articles/1662244.html)

可以新建一个类封装Hibernate需要动态配置的属性，然后根据不同的环境返回不同的配置：

	@Configuration
	public class HibernateConfig {
	
	
	    public static class HibernatePropertiesConfig {
	        private final String dialect;
	        private final String hbm2ddl_auto;
	
	        public HibernatePropertiesConfig(String dialect, String hbm2ddl_auto) {
	            this.dialect = dialect;
	            this.hbm2ddl_auto = hbm2ddl_auto;
	        }
	
	        public String getDialect() {
	            return dialect;
	        }
	
	        public String hbm2ddl_auto() {
	            return hbm2ddl_auto;
	        }
	    }
	
	    @Bean
	    LocalSessionFactoryBean factoryBean(DataSource dataSource, HibernatePropertiesConfig propertiesConfig) {
	        LocalSessionFactoryBean sfb = new LocalSessionFactoryBean();
	        sfb.setDataSource(dataSource);
	        sfb.setAnnotatedPackages("com.stephen.bangbang.domain");
	        sfb.setPackagesToScan("com.stephen.bangbang.domain");
	        Properties properties = new Properties();
	        properties.put("hibernate.dialect", propertiesConfig.getDialect());
	        properties.put("hibernate.hbm2ddl.auto", propertiesConfig.hbm2ddl_auto());
	        properties.put("hibernate.show_sql", "true");
	        properties.put("hibernate.format_sql", "true");
	        sfb.setHibernateProperties(properties);
	        return sfb;
	    }
	
	    @Bean
	    @Profile("production")
	    public HibernatePropertiesConfig productionDialect() {
	        return new HibernatePropertiesConfig("org.hibernate.dialect.MySQL57Dialect", "update");
	    }
	
	    @Bean
	    @Profile("dev")
	    public HibernatePropertiesConfig testDialect() {
	        return new HibernatePropertiesConfig("org.hibernate.dialect.H2Dialect", "create");
	    }
	}
## 配置事务管理
可以将Hibernate的事务交由Spring进行统一管理，关于Spring事务的更多内容，参见：[全面分析 Spring 的编程式事务管理及声明式事务管理](https://www.ibm.com/developerworks/cn/education/opensource/os-cn-spring-trans/index.html)

如果是使用Java类配置的话，则在配置类上加上**@EnableTransactionManagement**注解，并提供Spring为Hibernate提供的事务管理器即可：

	@Configuration
	@EnableTransactionManagement
	public class HibernateConfig {
	
	    ...
	
	    @Bean
	    public HibernateTransactionManager transactionManager(SessionFactory sessionFactory) {
	        HibernateTransactionManager transactionManager = new HibernateTransactionManager();
	        transactionManager.setSessionFactory(sessionFactory);
	        return transactionManager;
	    }
	}
如果使用xml配置方法，则需要使用：

	<tx:annotation-driven transaction-manager="transactionManager"/>
注意**transactionManager**是默认的名字，如果为**HibernateTransactionManager**指定了不同名字的话，则需要在注解上或者在**&lt;tx:annotation-driven&gt;**节点上指定。

# 使用
## 得到特定于线程的持久化上下文
接下来就可以在DAO类当中使用**SessionFactory**，需要使用持久化上下文时，推荐的做法是调用**SessionFactory#getCurrentSession()**方法：

	protected Session getCurrentSession() {
        return sessionFactory.getCurrentSession();
    }
因为此方法会获得特定于线程的**Session**，当当前线程没有Session时则创建一个并返回。此方法获得持久化上下文还可以与Spring的事务管理无缝对接，当一个事务提交时，当前线程的Session会自动关闭。
## 生成DTO类
有时候不需要全部返回实体类，为了方便客户端和服务端的消息通信，有时候需要DTO层，Hibernate可以通过JPQL来生成DTO类或者通过**ResultTransformer**来生成DTO类。

直接生成DTO:

    Query<TaskSnapshot> snapshotQuery = session.createQuery("select new com.stephen.bangbang.dto.TaskSnapshot(h) from HelpingTask h where h.user.id = :userId order by h.id desc").setParameter("userId", userId);
    List<TaskSnapshot> snapshots = snapshotQuery.getResultList();

ResultTransformer:

	query.setResultTransformer(new AliasToBeanResultTransformer(TaskSnapshot.class));

⚠️**注意：**上述**ResultTransformer**的使用方式在Hibernate5.2版本中已经被弃用了，目前我还没有找到解决的办法，官方文档只说setResultTransformer方法被弃用了没给出新的解决方案……所以第二种解决方案还是在5.2之前的版本中使用吧。

使用**AliasToBeanResultTransformer**时转换的类应该具有与查询语句中别名相同的属性或者拥有设置方法。

## 分页技术
分页是非常常见的一种技术，主要有两种实现方案：偏移量分页和搜寻分页。
### 偏移量分页
偏移量分页适用于这种UI:
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-07%20%E4%B8%8B%E5%8D%888.55.58.png)
实现方式也很容易，通过**setFirstResult**和**setMaxResults**方法即可实现，不过此种方法有一定的限制：

- 如果数据量十分巨大，那偏移量分页的性能将难以接受，此时需要进行一系列的优化，有关数据库分页的优化方法，参见：[MySQL分页优化](http://imysql.com/2014/07/26/mysql-optimization-case-paging-optimize.shtml)
- 偏移量分页会产生不正确的结果，例如用户查看第一页的时候其余的用户往集合当中插入了数据，此时读取数据的用户跳到第二页有可能看到第一页已经看到的数据，这种错误在网页上还可以接受，因为网页大多只展示一页的结果，可对于移动端APP，此种错误将难以接受，因为移动端大多数是通过上拉加载来请求数据的，第一页的数据通常还在用户的可视范围之内。

综上两个原因，偏移量分页适用于数据量小且不容易变更的数据。

### 搜寻分页
搜寻分页解决了偏移量分页面对的两个问题，其基本思路是：假如读取数据集合时需要根据某些属性进行排序(通常情况下也是如此)，那么可以通过传入的上一页的最后一个数据的被用于排序的属性来加载下一页。此种方法适用于移动端列表上拉加载的那种UI界面，不适合根据指定页数跳到相应界面的情况。

通过一个项目中的小例子来看一下搜寻分页的使用：

	@Repository
	@Transactional(isolation = Isolation.SERIALIZABLE)
		public class TaskRepositoryImpl extends BaseRepositoryImpl implements TaskRepository {
	
	    @Inject
	    public TaskRepositoryImpl(SessionFactory sessionFactory) {
	        super(sessionFactory);
	    }
	
	    @Override
	    public TasksResponse findAllTasks(Long lastTaskId, int number) {
	        Session session = getCurrentSession();
	        int totalPage = getPageCount(number);
	        int currentPage;
	        
	        // 为0则请求第一页
	        if (lastTaskId == 0) {
	            currentPage = 1;
	            if (currentPage > totalPage)
	                currentPage = totalPage;
	            Query<TaskSnapshot> snapshotQuery = session.createQuery("select new com.stephen.bangbang.dto.TaskSnapshot(h) from HelpingTask h order by h.id desc").setMaxResults(number);
	            List<TaskSnapshot> snapshots = snapshotQuery.getResultList();
	            return new TasksResponse(new Pagination(currentPage, totalPage), snapshots);
	        } else {
	            currentPage = getCurrentPage(lastTaskId, number);
	            if (currentPage > totalPage) {
	                currentPage = totalPage;
	                return new TasksResponse(new Pagination(currentPage, totalPage), null);
	            }
	            Query<TaskSnapshot> snapshotQuery = session.createQuery("select new com.stephen.bangbang.dto.TaskSnapshot(h) from HelpingTask h where h.id < :lastId order by h.id desc").setMaxResults(number);
	            List<TaskSnapshot> snapshots = snapshotQuery.getResultList();
	            return new TasksResponse(new Pagination(currentPage, totalPage), snapshots);
	        }
	    }
	
	    private int getPageCount(int numberPerPage) {
	        Session session = getCurrentSession();
	        Query<Long> query = session.createQuery("select count(h) from HelpingTask h");
	        long total = query.getSingleResult();
	        return countPage((int) total, numberPerPage);
	    }
	    
	    private int getCurrentPage(Long lastTaskId, int numberPerPage) {
	        Session session = getCurrentSession();
	        Query<Long> query = session.createQuery("select count(h) from HelpingTask h where h.id >= :lastId").setParameter("lastId", lastTaskId);
	        long count = query.getSingleResult() + 1;
	        return countPage((int) count, numberPerPage);
    	}
	
	    private int countPage(int total, int numberPerPage) {
	        if (total % numberPerPage == 0) {
	            return total / numberPerPage;
	        }
	
	        return total / numberPerPage + 1;
	    }

	}
Pagination类只是对分页信息的简单封装，包括用户所在的页数和总页数，总页数可以很容易被计算出来，至于得出用户所在的界面，则需要知道排序规则，上一页中最后一个数据的排序属性值，以及一页要多少个数据，三项都知道则可以计算出在最后一个数据之前有多少个数据，得出来的结果与每页数据数一比即可得出用户所在页数。

好了先讲这么多，以后再继续补充！
	
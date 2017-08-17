---
title: Bitronix with Hibernate5
date: 2017-07-19 17:15:19
tags: [数据库,ORM,Hibernate,Java Web,配置]
categories: ORM

---
记录一下开源JTA事务管理器Bitronix与Hibernate的集成
<!--more-->

一般情况下，JTA是和J2EE容器绑定起来的。要想在独立的Java环境下使用JTA，则要用第三方库。笔者采用Bitronix(BTM2)。本篇讲述如何整合BTM2 + Hibernate5开发环境。

# 创建jndi.properties文件
第一步，是要在classpath下建立名为jndi.properties的文件，最好放在**resource**文件目录下，整个项目结构如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-19%20%E4%B8%8B%E5%8D%885.30.51.png)
在jndi.properties文件里面写入：

	java.naming.factory.initial=bitronix.tm.jndi.BitronixInitialContextFactory
# 创建数据源
需要用PoolingDataSource建立数据源，并在persistence.xml文件当中引用它。

main函数：

	public static void main(String[] args) throws SystemException,
            NotSupportedException,
            HeuristicRollbackException,
            HeuristicMixedException,
            RollbackException {
        PoolingDataSource ds = new PoolingDataSource();
        ds.setMinPoolSize(1);
        ds.setMaxPoolSize(10);
        ds.setPreparedStatementCacheSize(10);
        // 此处设置的名称应该和persistence.xml文件中<jta-data-source>节点一致
        ds.setUniqueName("MySQLDataSource");

        ds.setIsolationLevel("READ_COMMITTED");
        ds.setAllowLocalTransactions(true);

        ds.setClassName("bitronix.tm.resource.jdbc.lrc.LrcXADataSource");
        ds.getDriverProperties().put("url", "jdbc:mysql://localhost:3306/hibernate_test");
        ds.getDriverProperties().put("driverClassName", "com.mysql.jdbc.Driver");
        ds.getDriverProperties().put("user", "root");
        ds.getDriverProperties().put("password", "xxxxxxx");

        ds.init();
        
        ...
    }
persistence.xml:

	<persistence-unit name="Hibernate">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
        <jta-data-source>MySQLDataSource</jta-data-source>
        <class>com.stephen.hibernatelearning.Item</class>
        <properties>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hbm2ddl.auto" value="create"/>
            <property name="hibernate.transaction.jta.platform" 
                      value="org.hibernate.service.jta.platform.internal.BitronixJtaPlatform" />
        </properties>
    </persistence-unit>
注意persistence.xml中最后的属性**hibernate.transaction.jta.platform**，以及**&lt;jta-data-source&gt;**节点中的值应该和刚才**PoolingDataSource#setUniqueName()**中设置的值对应起来。
# JNDI查找UserTransaction
**UserTransaction**是JTA的核心，需要通过jndi找到它。可以采用**InitialContext**类寻找：

	public static void main(String[] args) throws SystemException,
            NotSupportedException,
            HeuristicRollbackException,
            HeuristicMixedException,
            RollbackException, NamingException {
        ...
        
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("Hibernate");

        Context context = new InitialContext();
        UserTransaction tx = (UserTransaction) context.lookup("java:comp/UserTransaction");
        
        ...
    }
接下来就能愉快的使用了：

	public static void main(String[] args) throws SystemException,
            NotSupportedException,
            HeuristicRollbackException,
            HeuristicMixedException,
            RollbackException, NamingException {
        ...

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("Hibernate");

        Context context = new InitialContext();
        UserTransaction tx = (UserTransaction) context.lookup("java:comp/UserTransaction");

        tx.begin();
        EntityManager em = emf.createEntityManager();
        em.persist(new Item("T"));
        tx.commit();
        em.close();
        emf.close();
        ds.close();
        TransactionManagerServices.getTransactionManager().shutdown();
    }
可以进行一定的封装，这里我就简单的封装一下(参考**《Hibernate实战》**示例代码)：

	public class TransactionManagerSetUp {
	    private static final String DATASOURCE_NAME = "MySQLDataSource";
	
	    private PoolingDataSource ds;
	    private Context context;
	
	    public TransactionManagerSetUp(String url) {
	        ds = new PoolingDataSource();
	        ds.setMinPoolSize(1);
	        ds.setMaxPoolSize(10);
	        ds.setPreparedStatementCacheSize(10);
	        ds.setUniqueName(DATASOURCE_NAME);
	
	        ds.setIsolationLevel("READ_COMMITTED");
	        ds.setAllowLocalTransactions(true);
	
	        ds.setClassName("bitronix.tm.resource.jdbc.lrc.LrcXADataSource");
	        ds.getDriverProperties().put("url", url);
	        ds.getDriverProperties().put("driverClassName", "com.mysql.jdbc.Driver");
	        ds.getDriverProperties().put("user", "root");
	        ds.getDriverProperties().put("password", "xxxxxx");
	
	        ds.init();
	    }
	
	    public Context getNamingContext() {
	        if (context == null)
	            try {
	                context = new InitialContext();
	            } catch (NamingException e) {
	                e.printStackTrace();
	            }
	        return context;
	    }
	
	    public UserTransaction getTransaction() {
	        try {
	            return (UserTransaction) (getNamingContext().lookup("java:comp/UserTransaction"));
	        } catch (NamingException e) {
	            throw new RuntimeException(e);
	        }
	    }
	
	    public void rollback() {
	        UserTransaction tx = getTransaction();
	        try {
	            if (tx.getStatus() == Status.STATUS_ACTIVE
	                    || tx.getStatus() == Status.STATUS_MARKED_ROLLBACK) {
	                tx.rollback();
	            }
	        } catch (SystemException e) {
	            e.printStackTrace();
	        }
	    }
	
	    public void stop() {
	        ds.close();
	        TransactionManagerServices.getTransactionManager().shutdown();
	    }

	}
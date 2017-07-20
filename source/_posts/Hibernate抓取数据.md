---
title: Hibernate抓取数据
date: 2017-07-20 12:39:26
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
本篇主要介绍延迟加载和急加载，抓取策略和配置文件，集中于Hibernate对数据的抓取。

# 延迟加载
如果使用延迟加载策略映射所有的关联和集合，那么Hibernate会在你调用访问器方法时加载数据。实现延迟加载的基本实现方式是代理。
## 实体代理
当调用**EntityManager#getReference()**时，会生成实体类的相应代理类：

	EntityManager em = EMF.createEntityManager();
    Item item = em.getReference(Item.class, 1L);
此时生成的是Item类的代理子类，因此在equals()方法中不能够使用代理的getClass()方法进行比较，应使用**instanceof**。为了能够生成正确的代理子类，应该保证实体类有一个公共的或者受保护的无参构造函数，同时实体类中的方法不应该被final修饰。

**调用代理类上的任何非标识符获取方法都会触发代理的初始化并访问数据库。**即对于前面的示例，调用item.getId()不会发生代理的初始化，但是item.getName()则会发生初始化并访问数据库。

当在**@ManyToOne**和**@OneToOne**注解中开启延迟加载时，对应的实体类实例将被延迟加载，即不去查询实体类对用的表。
## 延迟持久化集合
对于集合，其默认的抓取策略是延迟加载。对于一下的代码：

	EntityManager em = EMF.createEntityManager();
    Item item = em.find(Item.class, 1L);
    Set<Bid> bids = item.getBids();
bids是未初始化(还未遍历该集合)的集合，Hibernate会采用**org.hibernate.collection.internal.PersistentSet**来包装该集合。此集合可以检测到访问它们的时间并在该时间内加载数据。只要开始遍历集合，就会加载所有的出价：

	// SELECT * FROM BID WHERE ITEM_ID = ...
    Bid firstBid = bids.iterator().next();
Hibernate为集合提供了专有设置，可以尽量避免加载整个集合到内存中：

	@OneToMany(mappedBy = "item", cascade = CascadeType.PERSIST)
    @org.hibernate.annotations.LazyCollection(
            LazyCollectionOption.EXTRA
    )
    protected Set<Bid> bids = new HashSet<>();
使用EXTRA选项，集合就会在某些时刻避免不必要的初始化工作，如调用item.getBids().size()此时会执行一个SELECT COUNT() SQL查询。在所有的额外延迟集合上都对isEmpty()和contains()方法执行类似查询。当调用add()方法时，延迟的Set会使用简单查询来避免重复值。当调用get(index)方法时，额外的延迟List只会加载一个元素(前提是开启**@OrderColumn**)。
# 关联和集合的急加载
有时候需要指定关联实体实例和集合的急加载，希望确保其在内存中可用，而不需要额外的数据库访问。

例如，如果希望在item实例处于分离状态下访问其中的user实例。当持久化上下文关闭的时候，延迟加载就会不可用。此时试图初始化user实=实例的代理会得到一个**LazyInitializationException**异常。为了让其在分离状态下可以访问，可以在持久化状态时手动加载，也可以指定**FetchType.EAGER**设置成急加载。

对于集合来说，急加载并非一个好的加载策略。如果急加载多个集合，还会出现笛卡尔积的问题，对于集合尽可能使用默认的延迟加载策略。
# 选取抓取策略
如果采用推荐的延迟加载策略，可能会造成很多的sql语句，产生n+1查询问题。如果使用急加载，将大块的数据加载到内存中，可能会产生笛卡尔积的问题。
## n+1查询问题
### 描述
假定有以下查询：

	List<Item> items = session.createQuery("select i from Item i").getResultList();
        
    for (Item i : items) {
        System.out.println(i.getUser().getName());
    }
其中Item中的user属性被设置成延迟加载的，那么假设集合的大小为n，那么总共需要执行n+1个select语句，但最少只需要一句select语句。
### 解决方案
#### 批量预加载数据
此解决方案的基本工作流程为：初始化一个代理，可以同时初始化多个代理。批量预加载数据的设置：

	@Entity
	@org.hibernate.annotations.BatchSize(size = 10)
	public class User {
	    ...
	}
这样在上面描述的例子当中，循环中执行一次i.getUser().getName()初始化一个User代理，同时就可以初始化10个在持久化上下文中的User类代理，直至持久化上下文中没有相应的代理为止。

批量预处理加载还可以用于集合：

	@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @org.hibernate.annotations.BatchSize(size = 10)
    protected Set<Item> items = new HashSet<>();
这样通过调用**user.getItems().iterator().next()**初始化一个集合代理时，便会同时有10个集合代理被初始化。

批量预加载数据是一种盲目优化算法。
#### 子查询
子查询目前为止只能应用到集合上：

	@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @org.hibernate.annotations.Fetch(
            FetchMode.SUBSELECT
    )
    protected Set<Item> items = new HashSet<>();
这样设置后一个集合代理被初始化后，持久化上下文中所有的User类实例的集合代理都将被初始化。
## 笛卡尔积问题
### 描述
当采用急加载策略同时加载两个集合时，因为所产生的sql语句是跨多个表的JOIN语句，所以会在底层数据库中产生**n\*m**个行数的结果集，Hibernate再对此结果集进行压缩。
### 解决方案
#### 采用多个SELECT语句
可以为一个集合的加载配置一个单独的SELECT语句，而不用JOIN语句加载所有的集合：

	@OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
    @org.hibernate.annotations.Fetch(
            FetchMode.SELECT
    )
    protected Set<Item> items = new HashSet<>();

    @OneToMany(fetch = FetchType.EAGER)
    @org.hibernate.annotations.Fetch(
            FetchMode.SELECT
    )
    protected List<Image> images = new ArrayList<>();
这样加载一个User实例时，会产生三个select语句。
#### 动态急抓取
可以通过在JPQL语句当中添加**join fetch**关键字强制进行急抓取，即便配置的抓取策略是**FetchType.LAZY**：

	// 加载实体
	List<Item> items = session.createQuery("select i from Item i join fetch i.user").getResultList();
	// 加载集合
	List<Item> items = session.createQuery("select i from Item i left join fetch i.bids").getResultList();
注意**left join fetch**是一个左外连接，因为即便一个item没有出价也应该被加载。**join fetch**是一个内连接。
# FetchProfile
可以声明Hibernate自带的配置文件，一般是包级别的定义：

	@org.hibernate.annotations.FetchProfiles(
        @FetchProfile(
                name = "ITEM_JOIN_USER",
                fetchOverrides = @FetchProfile.FetchOverride(
                        entity = Item.class,
                        association = "user",
                        mode = FetchMode.JOIN
                )
        )
	)
	package com.stephen.hibernatepractice;
启用**FetchProfile**:

	session.enableFetchProfile("ITEM_JOIN_USER");
	// 此时user是急抓取的。
    session.find(Item.class, 1000L);
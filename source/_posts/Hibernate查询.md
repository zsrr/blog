---
title: Hibernate查询
date: 2017-07-29 10:15:58
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
本篇致力于Hibernate对数据的查询，主要有JPQL和Criteria两种方式，以前者为主。

# 选择
选择所有的实例：

		Session session = em.unwrap(Session.class);
        Query query = session.createQuery("select i from Item i");
        List<Item> items = query.getResultList();

        for (Item item: items) {
            System.out.println(item);
        }

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Item> criteriaQuery = cb.createQuery(Item.class);
        criteriaQuery.select(criteriaQuery.from(Item.class));
        TypedQuery<Item> typedQuery = em.createQuery(criteriaQuery);
        List<Item> itemList = typedQuery.getResultList();
        for (Item item : itemList) {
            System.out.println(item);
        }
## 多态查询
对于具有继承关系的实体类来说，可以采用多态查询：

		Query<Super> superQuery = session.createQuery("select s from Super s");
        Query<Derived1> derived1Query = session.createQuery("select d1 from Derived1 d1");
        Query<Derived2> derived2Query = session.createQuery("select d2 from Derived2 d2");
指定具体的类：

	    Query<Super> superQuery = session.createQuery("select s from Super s where type(s) = Derived1");
        List<Super> derived1s = superQuery.getResultList();
        for (Super s : derived1s) {
            System.out.println(s.getClass().getName());
        }

	    Query<Super> superQuery = session.createQuery("select s from Super s where type(s) in :types");
        superQuery.setParameter("types", Arrays.asList(Super.class, Derived1.class));
        List<Super> derived1s = superQuery.getResultList();
        for (Super s : derived1s) {
            System.out.println(s.getClass().getName());
        }


# 限制
限制对应的是**SQL**中的**where**子句：

	    Query<Item> query = session.createQuery("select i from Item i where i.name = :name");
        query.setParameter("name", "Java Persistence With Hibernate");
        Item item = query.getSingleResult();
        System.out.println(item);

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Item> criteriaQuery = cb.createQuery(Item.class);
        Root<Item> root = criteriaQuery.from(Item.class);
        criteriaQuery.select(root).where(cb.equal(root.get("name"), cb.parameter(String.class, "itemName")));
        javax.persistence.Query itemQuery = em.createQuery(criteriaQuery).setParameter("itemName", "Java Persistence With Hibernate");
        Item item1 = (Item) itemQuery.getSingleResult();
        System.out.println(item1);

对于日期来进行比较的话，可以用JDBC转义语法：

	where i.auctionEnd = {d '2017-7-29'}
## 比较表达式
### between
利用between关键字可以进行范围内查找：

		Query<Bid> query = session.createQuery("select b from Bid b where b.amount between 90 and 100");
        List<Bid> bids = query.getResultList();
        for (Bid bid : bids) {
            System.out.println(bid);
        }

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Bid> criteriaQuery = cb.createQuery(Bid.class);
        Root<Bid> root = criteriaQuery.from(Bid.class);
        criteriaQuery.select(root).where(cb.between(root.<BigDecimal>get("amount"), new BigDecimal(90), new BigDecimal(100)));
        javax.persistence.Query bidQuery = em.createQuery(criteriaQuery);
        List<Bid> bidList = bidQuery.getResultList();
        for (Bid bid: bidList) {
            System.out.println(bid);
        }
Criterial API实在是太冗杂了，之后不再展示，具体去查官方文档。

### > < <> =
很简单，第三个是不等于的意思:

	    Query<Bid> query = session.createQuery("select b from Bid b where b.amount <> 90");
        List<Bid> bids = query.getResultList();
        for (Bid bid : bids) {
            System.out.println(bid);
        }
需要注意的是和**null**的比较，SQL使用三元逻辑表示限制，任何与null值的比较返回的都是null而非true和false，因此要用is null|is not null来和null值进行比较。
### like通配符搜索
可以用like关键字来搜索匹配模式串的记录：

        Query<Item> query = session.createQuery("select i from Item i where i.name like '%sr'");
        List<Item> items = query.getResultList();
        for (Item item : items) {
            System.out.println(item);
        }
关于like的更多操作，参见：[SQL LIKE操作符](http://www.w3school.com.cn/sql/sql_like.asp)
不同的条件可以用and/or关联，太简单不说了。
## 使用集合的表达式
JPQL提供了对应的函数来检测集合：

        Query<Item> query = session.createQuery("select i from Item i where i.bids is not empty");
        List<Item> items = query.getResultList();
      
        Query<Item> query = session.createQuery("select i from Item i where size(i.bids) > 1");
        List<Item> items = query.getResultList();
        
        Query<Item> query = session.createQuery("select i from Item i where :item member of i.bids");
        List<Item> items = query.getResultList();

对于Map而言，可以使用key(), value(), entry()来操作：

        Query<Item> query = session.createQuery("select value(image) from Item i join i.images image where key(image) like '@.jpg'");
        List<Item> items = query.getResultList();
## 调用函数
可以在where子句中调用函数，对于有哪些函数，我就不在这里细谈，参见：[JPQL学习](http://www.jianshu.com/p/4a4410075bab)

⚠️**注意：**对于Hibernate不知道的函数，如果出现在where子句中，则Hibernate会将其传递到数据库底层，要想在原生JPA当中调用适用于特定数据库的函数，则应使用**function()**函数：

	select i from Item i where function('DATEDIFF', 'DAY', i.createdOn, i.auctionEnd) > 1
## 对查询结果进行排序
可以用**order by**关键字对结果进行排序，默认为升序：

	select i from Item i order by i.name [desc|asc]
可以对多个属性进行排序：
	
	select i from Item i order by i.name desc, id asc
### null的顺序
如果可为空的列出现在order by子句中的话，那么空值可能出现在结果集的最前面或者最后面，具体行为取决于DBMS，可以在Hibernate配置文件配置属性**hibernate.order_by.default_null_ordering**为none(默认),first,last来指定。
# 投影
有时候不需要将一个实例的全部属性加载，这个时候需要投影来查询部分属性。
## 实体和标量值的投影
对于以下查询：

        Query query = session.createQuery("select i, b from Item i, Bid b");
        List<Object[]> resultList = query.getResultList();
        Set<Item> itemSet = new HashSet<>();
        Set<Bid> bidSet = new HashSet<>();
        
        for (Object[] row : resultList) {
            if (row[0] instanceof Item) {
                itemSet.add((Item) row[0]);
            }
            
            if (row[1] instanceof Bid) {
                bidSet.add((Bid) row[1]);
            }
        }
        
        System.out.println(resultList.size());
        System.out.println(itemSet.size());
        System.out.println(bidSet.size());
若是投影多个实体类，得到的是两个集合的笛卡尔乘积，可以用HashSet对结果进行过滤。

以下查询也会有Object[]的集合：

	select u.id, u.username, u.address from User u
查询返回的Object[]第一个位置是Long类型，第二个是String类型，第三个是Address类型，选择出来的都是处于瞬时状态的实例，不会进行脏检查和更新。

在使用瞬时实例的场景中(例如web应用中的DTO)，可以使用动态实例化或者ResultTransformer来把结果转换成瞬时的DTO实例，在接下来的Spring+Hibernate整合篇讲。
## 选择唯一值
对于可能出现重复值的列，可以用distinct关键字进行过滤：

	select distinct u.username from User u
## 在投影中使用函数
除了在where子句中使用函数之外，也可以在select子句当中使用函数，此时Hibernate不可以将未知的函数传递到数据库的底层，要想用特定于数据库的函数，只能用function函数。同时也可以在投影中使用case/when表达式。
## 聚合函数
可以使用count(), max(), min(), avg()等聚合函数：

	select count(distinct u.name) from User u
此时返回的是一个Long值，对于以下查询：

	select min(b.amount), max(b.amount) from Bid b
此时返回的是一个Object[]数组。
## 分组
可以在JPQL中使用**GROUP BY**和**HAVING**子句，在select子句当中出现的任何属性除了聚集函数之外都应该在GROUP BY当中出现：

	select i.name, avg(b.amount) from Bid b join b.item i group by i.name
Hibernate分组存在限制，例如不能编写以下查询：

	select i, avg(b.amount) from Bid b join b.item i group by i
# 联结
联结是JPQL中最难理解的部分，首先应该知道了解sql中联结的概念：[深入理解sql中的四种连接](http://www.jb51.net/article/39432.htm)
## JPA联结选项
JPA提供了四个联结选项：

- 使用路径表达式的隐式连接。
- 在FROM子句当中使用Join关键字的普通联结。
- 在FROM子句当中使用Join Fetch进行动态急抓取。
- 在Where子句中使用theta风格联结。

接下来对四种方式挨个进行介绍。
## 隐式联结
在多对一／一对一关系当中，已经指定了相应的外键关联，因此在JPQL语句当中访问对应属性时会产生隐式关联：

	select b from Bid b where b.item.name like 'D%'
当执行此语句时，Hibernate会产生一个从Bid到Item表的Inner Join语句(也有可能是中间表)。隐式联结出现在多对一／一对一关联当中，不会出现在一对多关联，所以不能编写item.bids.amount > 10这种语句。

单路径表达式中可以使用多个联结：

	select b from Bid b where b.item.seller.username = 'zsr'
此语句创建了从Bid到Item到User表的关联。
## 显式联结
如上所述，隐式联结适合于对一关联，要想对对多关系进行联结，则需要使用显式联结：

	select i from Item i join i.bids b where b.amount > 10
以上查询语句建立了从Item到Bid表的内联结，内联结的条件是Item表中的主键和对应于Bid表中的外键相等，这里可以忽略不写。最后对结果集进行过滤，过滤的条件是Bid表中的出价大于10。

⚠️**注意：**由于是一对多关联，所以对连接表查询的结果映射到List<Item>会有重复项出现，可以使用Set进行过滤：

		Query query = session.createQuery("select i from Item i join i.bids b where b.amount > 0");
        List<Item> items = query.getResultList();
        Set<Item> items1 = new HashSet<>(items);
        System.out.println(items.size()); // 5
        System.out.println(items1.size()); // 2

也可以扩展联结on之后的条件，例如想要查找未出价或者有最小出价的商品：

	select i from Item i left join i.bids b on b.amount > 100
首先要有left join，这会将不具备出价的Item包含在查询的结果集当中；其次，上述语句on关键字出现的地方不能被where代替，on的语义是指：对满足两表主外键值相等的联结条件进行扩展，增加"Bid表中amount值应大于100"这一条件，而where的语义是联结条件不变，对结果集进行过滤，因此若把之前的on换成where，那left join将不会有任何效果，结果集中所有的Bid表中的部分都不为空。
## 动态急抓取
实体的集合属性默认是延迟加载的，可以通过编写JPQL来改变此行为：

	select i form Item i left join fetch i.bids
此时选出的每一个Item其bids集合都是经过初始化的，由于其本质上还是连接查询，因此会产生重复项，需要用Set集合进行去重或者使用distinct关键字：

	select distinct i form Item i left join fetch i.bids
动态急抓取需要注意下面几点：

- 不要为进一步限制或者投影将别名指定给抓取的关联实体或者集合。例如：left join fetch i.bids b where b.amount > 100，此查询是无效的。但是别名可以用来进一步抓取：left join fetch i.bids b join fetch b.bidder。
- 不应该抓取多个集合，否则会产生笛卡尔乘积问题。
- 如果急抓取一个集合，要注意重复项问题。
- 急抓取无法在数据库层面进行分页，例如对于select i from Item i left join fetch i.bids。如果对于此Query设置firstResult以及maxResults，则数据库无法对其进行分页。此时Hibernate会将所有的Item加载到内存中并执行分页，此行为可能导致性能瓶颈出现。

## theta风格联结
当两个表不是通过外键建立参照完整性约束时，就可以创建theta风格的关联，例如：

	select u, log from User u, LogRecord log where u.username = log.username
User表和LogRecord表之间没有建立任何的关联，此时返回的元组列表每一个元组对应的两个记录用户名相同。
## 比较标识符
JPA支持隐式标识符比较：

	select i, u from Item i, User u where i.seller = u and u.username like 'D%'

相当于以下theta风格的查询：

	select i, u from Item i, User u where i.seller.id = u.id and u.username like 'D%'

⚠️**注意**，以id结尾的路径表达式指向的是实体中@Id注解的属性，不管这个属性是否叫做id，因此将标识符属性命名为id可以减轻混淆。
# 子查询
JPA仅支持where子句的子查询，select和from子句的子查询不支持。

一个子查询通常会返回单行或者多行，返回单行的子查询通常会调用聚合函数：

	select u from User u where (select count(i) from Item i where i.seller = u) > 1
此内查询同时也是一个相关的子查询，用到了外部传来的u。
## 量化
常用的标准量词：

- ALL: 对于子查询结果集所有结果比较结果都为真。
- ANY: 对于子查询结果集中任意一个结果比较为真。
- EXITS: 子查询结果不为空。


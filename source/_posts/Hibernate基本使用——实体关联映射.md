---
title: Hibernate基本使用——实体关联映射
date: 2017-07-13 21:19:37
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
本篇主要讨论实体间关联的映射，包括一对一关联，一对多关联，多对多关联，三元关联。
# 一对一关联
创建一对一关联主要有以下几种方式：
## 共享主键
假设**User**类和**Address**类具有一对一关联，下面通过共享主键值来实现：

	@Entity
	public class Address {
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @Column(nullable = false)
	    protected String city;
	
	    @Column(nullable = false)
	    protected String zipCode;
	
	    @Column(nullable = false)
	    protected String street;
	}
	
	@Entity
	public class User {
	
	    @Id
	    protected Long id;
	
	    @Column(nullable = false)
	    String name;
	
	    public User() {
	
	    }
	
	    public User(long id, String name) {
	        this.id = id;
	        this.name = name;
	    }
	    
	    @OneToOne(fetch = FetchType.LAZY, optional = false)
	    @PrimaryKeyJoinColumn
	    protected Address address;
	}
	
观察生成的sql语句：

	create table Address (
       id bigint not null,
        city varchar(255) not null,
        street varchar(255) not null,
        zipCode varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    create table User (
       id bigint not null,
        name varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    alter table User 
       add constraint FK5sra0gqx5jpad8o7jsusigt9t 
       foreign key (id) 
       references Address (id)
对于User类，没有指定主键生成器，通过构造函数传入id来指定，此id一般来自于前面已经保存的Address实例的id。

注解**@OneToOne**标记一对一关联关系，可以只在一个类中映射，也可以在互相关联的两个类中分别映射另一个类。**FetchType.LAZY**指定延迟加载策略，**注意只有指定optional为false时此策略才能够正常的工作。**

在主函数中：

	public static void main(String[] args) {
        Session session = factory.openSession();
        Transaction tx = session.beginTransaction();
        Address address = new Address();
        address.city = "Lai Wu";
        address.street = "PengQuanDong";
        address.zipCode = "271100";
        session.persist(address);

        User user = new User(address.id, "zsr");
        user.address = address;
        session.persist(user);

        System.out.println(user.address.city);

        tx.commit();
        session.close();
        factory.close();
    }
首先要保存Address实例，要保证其id在保存之后可用，因此应该选择INSERT之前生成id的策略。然后为User实例指定id，因为指定optional为false，所以要再次指定User的address。

## 外主键生成器
可以用外主键生成器实现一对一的双向导航，需在一个类中指定mappedBy侧：

	@Entity
	public class User {
	
	    public User() {
	
	    }
	
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @Column(nullable = false)
	    protected String name;
	
	    @OneToOne(mappedBy = "user", cascade = CascadeType.PERSIST)
	    protected Address address;
	}

	@Entity
	public class Address {
	    @Id
	    @GenericGenerator(name = "foreignKeyGenerator",
	            strategy = "foreign",
	            parameters = @Parameter(name = "property", value = "user"))
	    @GeneratedValue(generator = "foreignKeyGenerator")
	    protected Long id;
	
	    @Column(nullable = false)
	    protected String city;
	
	    @Column(nullable = false)
	    protected String zipCode;
	
	    @Column(nullable = false)
	    protected String street;
	
	    @OneToOne(optional = false)
	    @PrimaryKeyJoinColumn
	    protected User user;
	
	    public Address() {
	
	    }
	
	    public Address(User user) {
	        this.user = user;
	    }
	}
mappedBy指定另一侧中成员的名字，**CascadeType.PERSIST**指定当保存User类实例时，顺便把其中的Address实例保存，之后会再谈论cascade行为。

Address类指定外主键生成器，同时在user属性中指定外键关联。

**注意在使用共享主键时，@OneToOne注解的optional选项一定要设置成false，否则不能生成外键关联，且延迟加载不能够正常的工作。**
## 使用外键联结列
可以指定一个表中的主键为另一个表中的普通列，通过外键关联，而不用共享主键：

	@Entity
	public class User {
	
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @Column(nullable = false)
	    protected String name;
	
	    @OneToOne(fetch = FetchType.LAZY,
	            // 此时此项不是必须的
	            optional = false,
	            cascade = CascadeType.PERSIST)
	    @JoinColumn(unique = true)
	    protected Address address;
	
	    // Getter and setter.
	}
生成的sql语句：

	create table Address (
       id bigint not null,
        city varchar(255) not null,
        street varchar(255) not null,
        zipCode varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    create table User (
       id bigint not null,
        name varchar(255) not null,
        address_id bigint not null,
        primary key (id)
    ) engine=MyISAM
    
    alter table User 
       add constraint UK_25yqck53dyy0k1q261ncjxmw3 unique (address_id)
       
    alter table User 
       add constraint FKlq0qkm58rh351bb84y4o5c447 
       foreign key (address_id) 
       references Address (id)
此时在User表中的address_id即为引向Address表的外键，并且有一个unique约束，表示一个Address实例只能被一个User实例引用。
## 使用一个联接表
也可以使用一个联接表来表示一对一关联：

	@OneToOne(fetch = FetchType.LAZY,
            cascade = CascadeType.PERSIST)
    @JoinTable(name = "USER_ADDRESS",
            joinColumns = @JoinColumn(name = "USER_ID"),
            inverseJoinColumns = @JoinColumn(name = "ADDRESS_ID", nullable = false, unique = true))
    protected Address address;
生成的sql语句：

	create table User (
       id bigint not null,
        name varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    create table USER_ADDRESS (
       ADDRESS_ID bigint not null,
        USER_ID bigint not null,
        primary key (USER_ID)
    ) engine=MyISAM
    
    alter table USER_ADDRESS 
       add constraint UK_6v1dwl45nt34pce4p648i1gtx unique (ADDRESS_ID)
    
    alter table USER_ADDRESS 
       add constraint FKocjk8pkyyoeqytsjegryu4ldl 
       foreign key (ADDRESS_ID) 
       references Address (id)
    
    alter table USER_ADDRESS 
       add constraint FKk1ny487mwjo57mxnmqpx6bvlw 
       foreign key (USER_ID) 
       references User (id)
比较简单，不再细说。
 
# 一对多／多对一关联
## 考虑使用集合映射
[上一篇文章](http://www.stephenzhang.me/2017/07/13/Hibernate基本使用——映射集合和实体关联/)介绍了集合的映射，在很多情况下一对多映射可以被嵌入类集合映射代替，且集合映射可以带来n种好处，例如，可以不用考虑作为实体带来的生命周期问题。

当一个类只和一个实体类相关联的时候，考虑用集合映射而非实体映射。
## 双向一对多／多对一映射
单向映射比较简单，可以被包含到双向映射中，单向不再谈，下面看双向映射：

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @OneToMany(fetch = FetchType.LAZY, mappedBy = "item")
	    protected Set<Bid> bids = new HashSet<>();
	
	    public Long getId() {
	        return id;
	    }
	
	    public String getName() {
	        return name;
	    }
	
	    public void setName(String name) {
	        this.name = name;
	    }
	}
	
	@Entity
	public class Bid {
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @ManyToOne(fetch = FetchType.LAZY, optional = false)
	    @JoinColumn(nullable = false, name = "ITEM_ID")
	    protected Item item;
	}
	
生成的sql语句：

	create table Bid (
       id bigint not null,
        ITEM_ID bigint not null,
        primary key (id)
    ) engine=MyISAM
    
    create table Item (
       id bigint not null,
        name varchar(255),
        primary key (id)
    ) engine=MyISAM
    
    alter table Bid 
       add constraint FK884gyguvo3jcbca0v78w95l3k 
       foreign key (ITEM_ID) 
       references Item (id)
即在多的那一侧生成一个关联的外键，比较简单规范，不再多说。
## 级联状态
可以通过级联状态来设置生命周期事件的传递。
### 级联保存
**CascadeType.PERSIST**表示当一个实体类实例被保存时，其里面的实体类属性实例也被对应保存。例如对Item类进行如下更改：

	@OneToMany(fetch = FetchType.LAZY, mappedBy = "item", cascade = CascadeType.PERSIST)
    protected Set<Bid> bids = new HashSet<>();
当一个Bid实例被添加到集合中去，那么当一个Item实例被保存时，前面被添加的Bid实例也会被保存。
### 级联删除
**CascadeType.REMOVE**表示当一个实体类实例被删除的时候，其里面的一对多属性实例被删除。对Item类进行如下更改：

	@OneToMany(fetch = FetchType.LAZY, mappedBy = "item", cascade = {CascadeType.PERSIST, CascadeType.REMOVE})
    protected Set<Bid> bids = new HashSet<>();
则当一个Item实例被删除时，其对应的所有出价也将被移除。

值得注意的是，此删除过程是低效的。因为Bid是一个实体类，删除必须遵守常规的生命周期，所以在级联删除的过程中，Hibernate将先加载全部的Bid实例，然后逐一删除，而不是通过单个delete语句。
### 孤立的删除
对Item进行如下更改：

	@OneToMany(fetch = FetchType.LAZY, mappedBy = "item", 
            cascade = {CascadeType.PERSIST, CascadeType.REMOVE}, 
            orphanRemoval = true)
    protected Set<Bid> bids = new HashSet<>();
则一个Bid实例从集合中移除，当事务提交时，Hibernate会认为此实例是孤立的，即没有任何其他实例在引用它，之后Hibernate执行delete语句将其删除。

孤立的删除是存在疑问的，如果Bid类被其他类引用的话，就会出现不一致性，因为Hibernate不会在数据库删除之后删除内存当中的引用。因此要确保“多”的那一侧只和“单”的那一侧相关联。
### 生成ON DELETE CASCADE架构
可以通过Hibernate自身的注解来生成SQL架构，让数据库完成级联操作。对Item类进行如下更改：

	@OneToMany(fetch = FetchType.LAZY, mappedBy = "item",
            cascade = CascadeType.PERSIST, CascadeType.REMOVE)
    @org.hibernate.annotations.OnDelete(action = OnDeleteAction.CASCADE)
    protected Set<Bid> bids = new HashSet<>();
这样在删除Item实例时，其对应的所有出价将被删除，不过还是要注意：**Hibernate不会在数据库删除之后删除对应的全局二级缓存，可能出现内存不一致性。**
## 一对多包
包具有集合和列表不具有的性能优势，因为包允许重复元素并具有无序性，因此往包中添加实例时，不必加载所有的元素。如果要映射很大的实体集合，那这是一个很重要的特性。
## 一对多的联接表实现
考虑一下上面所说的一对一实现，方法是将其中一个作为主键，一个作为外键并且添加unique约束。一对多也可以用联接表来实现，方法是把多的那一侧的实体主键作为新表的主键，单的那一侧的主键仅作为新表的外键。假设Buyer和Item拥有一对多关系：

	@Entity
	public class Buyer {
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    // 在此指定mappedBy
	    @OneToMany(fetch = FetchType.LAZY, mappedBy = "buyer")
	    protected Set<Item> items = new HashSet<>();
	}
	
	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @ManyToOne(fetch = FetchType.LAZY)
	    @JoinTable(name = "BUYER_ITEM", joinColumns = @JoinColumn(name = "ITEM_ID"),
	            inverseJoinColumns = @JoinColumn(nullable = false))
	    protected Buyer buyer;
	
	    ...
	}
生成的sql语句如下：

	create table BUYER_ITEM (
       buyer_id bigint not null,
        ITEM_ID bigint not null,
        primary key (ITEM_ID)
    ) engine=MyISAM
    
    alter table BUYER_ITEM 
       add constraint FKano0nqxwa2osjdiy5xe4k1ipy 
       foreign key (buyer_id) 
       references Buyer (id)
    
    alter table BUYER_ITEM 
       add constraint FKpumh10h4kwj5xtap4ukicm2yb 
       foreign key (ITEM_ID) 
       references Item (id)
**注意：@JoinTable注解出现在将要在主键将要在新表中担任主键的实体类中，且mappedBy不能和其同用。**

mappedBy的具体含义，参见：[hibernate的注解属性mappedBy详解](http://shenyuc629.iteye.com/blog/1681225)
## 可嵌入类的一对多关联
假设Address类是一个可嵌入类，其于Shipment类有一个一对多关联：

	@Embeddable
	public class Address {
	
	    public Address(String city, String zipCode, String street) {
	        this.city = city;
	        this.zipCode = zipCode;
	        this.street = street;
	    }
	
	    @Column(nullable = false)
	    protected String city;
	
	    @Column(nullable = false)
	    protected String zipCode;
	
	    @Column(nullable = false)
	    protected String street;
	
	    @OneToMany
	    @JoinColumn(nullable = false, name = "USER_ID")
	    protected Set<Shipment> shipments = new HashSet<>();
	
	    ...
	}
	
	@Entity
	public class User {
	
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @Column(nullable = false)
	    protected String name;
	
	    protected Address address;
	
	    ...
	}
生成的sql语句：

	create table User (
       id bigint not null,
        city varchar(255) not null,
        street varchar(255) not null,
        zipCode varchar(255) not null,
        name varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    create table Shipment (
       id bigint not null,
        USER_ID bigint not null,
        primary key (id)
    ) engine=MyISAM
    
    alter table Shipment 
       add constraint FKfl6pih1ijc27muej6rnild61k 
       foreign key (USER_ID) 
       references User (id)
可嵌入组件没有自己的标识符，因此名为USER_ID的外键指向的是User类的外键，同时，双向导航是不行的，因为可嵌入组件不能共享引用。

如果需要使关联关系可选并且不希望有非空的列，则可以使用联接表来表示一对多：

	@OneToMany
    @JoinTable(name = "DELIVERIES", joinColumns = @JoinColumn(name = "USER_ID"), 
            inverseJoinColumns = @JoinColumn(name = "SHIPMENT_ID"))
    protected Set<Shipment> shipments = new HashSet<>();
生成的sql语句：
	
	create table DELIVERIES (
       USER_ID bigint not null,
        SHIPMENT_ID bigint not null,
        primary key (USER_ID, SHIPMENT_ID)
    ) engine=MyISAM
    
    alter table DELIVERIES 
       add constraint FKtblkdrxvth5ewvq1nabv58eaj 
       foreign key (SHIPMENT_ID) 
       references Shipment (id)
    
    alter table DELIVERIES 
       add constraint FKdtyybuqlk3c5me626v72qyg 
       foreign key (USER_ID) 
       references User (id)
如果不指明**@JoinColumn**和**@JoinTable**，则在可嵌入组件的一对多关系当中，默认采用后者。
# 多对多关联
## 隐藏中间表
一般采用中间表的方式来实现多对多关联：

	@Entity
	public class Category {
	    @Id
	    @GeneratedValue(generator = Constants.CUSTOM_GENERATOR)
	    protected Long id;
	
	    @ManyToMany(cascade = CascadeType.PERSIST)
	    @JoinTable(name = "CATEGORY_ITEM",
	            joinColumns = @JoinColumn(name = "CATEGORY_ID"),
	            inverseJoinColumns = @JoinColumn(name = "ITEM_ID"))
	    protected Set<Item> items = new HashSet<>();
	}
	
	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    ...
	
	    @ManyToMany(mappedBy = "items")
	    protected Set<Category> categories = new HashSet<>();
	
	    ...
	}
如果要映射像List这样的集合，只能在一侧使用，如果两侧都是链表，则只能一侧被持久化。

上述中间表对于Java层是隐藏的，也可以自己亲自建立一个实体类，用作中间表。
## 中间实体多对多关联
可以采用一个Java类来表示中间表。此类与两端具有多对多关系的两个类都是一对多关系：

	@Entity
	@org.hibernate.annotations.Immutable
	public class CategorizedItem {
	
	    @Embeddable
	    private static class Id implements Serializable {
	        @Column(name = "CATEGORY_ID", nullable = false)
	        protected Long categoryId;
	
	        @Column(name = "ITEM_ID", nullable = false)
	        protected Long itemId;
	
	        public Id() {
	        }
	
	        public Id(Long categoryId, Long itemId) {
	            this.categoryId = categoryId;
	            this.itemId = itemId;
	        }
	
	        @Override
	        public boolean equals(Object o) {
	            if (this == o) return true;
	            if (o == null || getClass() != o.getClass()) return false;
	
	            Id id = (Id) o;
	
	            if (!categoryId.equals(id.categoryId)) return false;
	            return itemId.equals(id.itemId);
	        }
	
	        @Override
	        public int hashCode() {
	            int result = categoryId.hashCode();
	            result = 31 * result + itemId.hashCode();
	            return result;
	        }
	    }
	
	    @EmbeddedId
	    protected Id id = new Id();
	
	    @Temporal(TemporalType.TIMESTAMP)
	    @Column(updatable = false)
	    @NotNull
	    protected Date date = new Date();
	
	    @ManyToOne
	    @JoinColumn(name = "CATEGORY_ID", insertable = false, updatable = false)
	    protected Category category;
	
	    @ManyToOne
	    @JoinColumn(name = "ITEM_ID", insertable = false, updatable = false)
	    protected Item item;
	
	    public CategorizedItem(Category category, Item item) {
	        this.category = category;
	        this.item = item;
	        this.id.categoryId = category.id;
	        this.id.itemId = item.id;
	        category.getCategorizedItems().add(this);
	        item.getCategorizedItems().add(this);
	    }
	
	    public CategorizedItem() {
	    }
	}
复合主键的组成部分就是Category类和Item类的主键，在构造函数中被指定，并且在之后不能更改。设置insertable和updatable为false的原因是主键是在构造函数中被设置，作为主键的组成部分应该设置成只读。
# 三元关联
三个以上的实体关联成为三元关联。
## 组件三元关联
可以采用组件的多对一／一对多特性来设置三元关联：

	@Embeddable
	public class CategorizedItem {
	    @ManyToOne
	    // 用作主键
	    @JoinColumn(name = "ITEM_ID", nullable = false)
	    protected Item item;
	
	    @ManyToOne
	    @JoinColumn(name = "USER_ID")
	    protected User user;
	
	    @Override
	    public boolean equals(Object o) {
	        if (this == o) return true;
	        if (o == null || getClass() != o.getClass()) return false;
	
	        CategorizedItem that = (CategorizedItem) o;
	
	        return item != null ? item.equals(that.item) : that.item == null;
	    }
	
	    @Override
	    public int hashCode() {
	        return item != null ? item.hashCode() : 0;
	    }
	}
	
	@Entity
	public class Category {
	    ...
	
	    @ElementCollection
	    @CollectionTable(name = "CATEGORY_ITEM", joinColumns = @JoinColumn(name = "CATEGORY_ID"))
	    protected Set<CategorizedItem> categorizedItems = new HashSet<>();
	
	    ...
	}
生成的sql语句：

	create table CATEGORY_ITEM (
       CATEGORY_ID bigint not null,
        ITEM_ID bigint not null,
        USER_ID bigint,
        primary key (CATEGORY_ID, ITEM_ID)
    ) engine=MyISAM
要注意可嵌入类的超键是如何声明的，应该重写Item类的equals和hashCode方法。此时Item和Category是多对多关联，两个分别和User是多对多关联。
## 键／值三元关联
也可以使用键值操作来实现三元关联：

	@Entity
	public class Category {
	    ...
	
	    @ManyToMany(cascade = CascadeType.PERSIST)
	    @MapKeyJoinColumn(name = "ITEM_ID")
	    @JoinTable(name = "CATEGORY_ITEM",
	            joinColumns = @JoinColumn(name = "CATEGORY_ID"),
	            inverseJoinColumns = @JoinColumn(name = "USER_ID"))
	    protected Map<Item, User> itemUserMap = new HashMap<>();
	
	    ...
	}
sql语句：
	
	create table CATEGORY_ITEM (
       CATEGORY_ID bigint not null,
        USER_ID bigint not null,
        ITEM_ID bigint not null,
        primary key (CATEGORY_ID, ITEM_ID)
    ) engine=MyISAM
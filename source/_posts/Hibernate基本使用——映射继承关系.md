---
title: Hibernate基本使用——映射继承关系
date: 2017-06-28 20:46:58
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
本篇主要集中于Hibernate对于Java类继承的处理方式
<!--more-->

回顾一下之前JPA处理继承类的方式，分成单表策略，合并表策略，一类一表策略，Hibernate也提供了类似的形式(JPA详见[Java ORM解决方案](http://www.stephenzhang.me/2017/04/14/Java-ORM%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/))。

# 每个带有联合的具体类使用一个表
TABLE_PER_CLASS策略的含义为：超类使用一个表，每个具体的字类单独使用一个表，并在表中复制超类列，下面给出一个例子：

	@Entity
	@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
	public class Super {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @NotNull
	    @Column(nullable = false)
	    protected String name;
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public String getName() {
	        return name;
	    }
	}
	
	@Entity
	public class Derived1 extends Super {
	    
	    @Column
	    protected String extra_derived1;
	
	    public String getExtra_derived1() {
	        return extra_derived1;
	    }
	
	    public void setExtra_derived1(String extra_derived1) {
	        this.extra_derived1 = extra_derived1;
	    }
	}
	
	@Entity
	public class Derived2 extends Super {
	    
	    @Column
	    protected String extra_derived2;
	
	    public String getExtra_derived2() {
	        return extra_derived2;
	    }
	
	    public void setExtra_derived2(String extra_derived2) {
	        this.extra_derived2 = extra_derived2;
	    }
	}
	
	    public static void main(String[] args) {
	        Session session = factory.openSession();
	        Transaction tx = session.beginTransaction();
	        Super s = new Super();
	        s.setName("zsr");
	        session.persist(s);
	        Derived1 d1 = new Derived1();
	        d1.setName("zsr1");
	        d1.setExtra_derived1("This is extra1");
	        session.persist(d1);
	        Derived2 d2 = new Derived2();
	        d2.setName("zsr2");
	        d2.setExtra_derived2("This is derived2");
	        session.persist(d2);
	        List<Super> supers = session.createQuery("select s from Super s").getResultList();
	        for (Super s_ : supers) {
	            System.out.println(s_.getName());
	        }
	        tx.commit();
	        factory.close();

    }
生成的sql语句如下：

	create table Derived1 (
       id bigint not null,
        name varchar(255) not null,
        extra_derived1 varchar(255),
        primary key (id)
    ) engine=MyISAM
    
    create table Derived2 (
       id bigint not null,
        name varchar(255) not null,
        extra_derived2 varchar(255),
        primary key (id)
    ) engine=MyISAM
    
    create table Super (
       id bigint not null,
        name varchar(255) not null,
        primary key (id)
    ) engine=MyISAM
    
    select
        super0_.id as id1_7_,
        super0_.name as name2_7_,
        super0_.extra_derived1 as extra_de1_2_,
        super0_.extra_derived2 as extra_de1_3_,
        super0_.clazz_ as clazz_ 
    from
        ( select
            id,
            name,
            null as extra_derived1,
            null as extra_derived2,
            0 as clazz_ 
        from
            Super 
        union
        select
            id,
            name,
            extra_derived1,
            null as extra_derived2,
            1 as clazz_ 
        from
            Derived1 
        union
        select
            id,
            name,
            null as extra_derived1,
            extra_derived2,
            2 as clazz_ 
        from
            Derived2 
    ) super0_
可以看到，当从数据库中检索Super类的实例时，Hibernate会对字类的表执行union操作。

# 每个类层次结构使用一个表
**InheritanceType.SINGLE_TABLE**表示的是将每个类层次结构映射到一个表中，即父类和字类映射到同一个表中，同时增加一个识别列来标示具体是哪个类，具体的例子如下：

	@Entity
	@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
	@DiscriminatorColumn(name = "TYPE")
	public class Super {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @NotNull
	    @Column(nullable = false)
	    protected String name;
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public String getName() {
	        return name;
	    }
	}
	
	@Entity
	@DiscriminatorValue("DERIVED1")
	public class Derived1 extends Super {
	
	    @Column
	    protected String extra_derived1;
	
	    public String getExtra_derived1() {
	        return extra_derived1;
	    }
	
	    public void setExtra_derived1(String extra_derived1) {
	        this.extra_derived1 = extra_derived1;
	    }
	}
	
	@Entity
	@DiscriminatorValue("DERIVED2")
	public class Derived2 extends Super {
	
	    @Column
	    protected String extra_derived2;
	
	    public String getExtra_derived2() {
	        return extra_derived2;
	    }
	
	    public void setExtra_derived2(String extra_derived2) {
	        this.extra_derived2 = extra_derived2;
	    }
	}
	
查看一下生成的sql语句：

	create table Super (
       TYPE varchar(31) not null,
        id bigint not null,
        name varchar(255) not null,
        extra_derived1 varchar(255),
        extra_derived2 varchar(255),
        primary key (id)
    ) engine=MyISAM
    
当查询某个具体类的实例时，则可根据TYPE列来检索。

# 每个带有联结的子类使用一个表
上面使用一个表的方法有一点反规范化，会让数据库难以维护，规范化的做法是让每个子类建立一个表格，然后设置外键引回到父类表，用法如下：

	create table Derived1 (
       extra_derived1 varchar(255),
        DERIVED1_KEY bigint not null,
        primary key (DERIVED1_KEY)
    ) engine=MyISAM
    
    create table Derived2 (
       extra_derived2 varchar(255),
        DERIVED2_KEY bigint not null,
        primary key (DERIVED2_KEY)
    ) engine=MyISAM
    
    alter table Derived1 
       add constraint FK9ra02o281myx52c8wogqrgwxt 
       foreign key (DERIVED1_KEY) 
       references Super (id)
    
    alter table Derived2 
       add constraint FKp55sbehplgk2hrhybl2phcojs 
       foreign key (DERIVED2_KEY) 
       references Super (id)
       
当对父表进行查询时，会对子类的两个表进行连接操作：

	select
        super0_.id as id1_7_,
        super0_.name as name2_7_,
        super0_1_.extra_derived1 as extra_de1_2_,
        super0_2_.extra_derived2 as extra_de1_3_,
        case 
            when super0_1_.DERIVED1_KEY is not null then 1 
            when super0_2_.DERIVED2_KEY is not null then 2 
            when super0_.id is not null then 0 
        end as clazz_ 
    from
        Super super0_ 
    left outer join
        Derived1 super0_1_ 
            on super0_.id=super0_1_.DERIVED1_KEY 
    left outer join
        Derived2 super0_2_ 
            on super0_.id=super0_2_.DERIVED2_KEY

# 对于策略的选择
选择合适的策略是至关重要的，下面介绍一些实际的准则。

如果不需要多态关联或者查询，则可以使用**TABLE_PER_CLASS**策略，换句话说，只要不经常执行**SELECT s FROM Super s**语句(对多表执行union操作)，尽量使用此策略。

如果确实需要多态关联，并且子类拥有较少的属性，则可以采取反规范化的做法采用SINGLE_TABLE策略。

当子类拥有很多的属性时，单表策略将使得数据库难以维护，此时可以采取规范化的做法建立外键关联。
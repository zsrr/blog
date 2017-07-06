---
title: Hibernate基本使用——实体映射
date: 2017-06-19 19:44:51
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---

本篇记录Hibernate实体映射的基础，采用IntelliJ Idea搭建Hibernate项目，具体如何搭建参见：[IntelliJ IDEA Hibernate 从入门到填坑](http://www.jianshu.com/p/50e0a7a28b53)

# 实体类和映射
一个实体类代表着数据库中的一个表，用**@Entity**进行标记，同时，一个持久化类内部要有一个id(主键)，用@Id进行注解，关于主键如何选择，参见：[数据库中主键的选择和使用](http://www.cnblogs.com/chuncn/archive/2009/04/22/1440901.html)。如下所示：

	@Entity
	public class User {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    public Long getId() {
	        return id;
	    }
	}
将**@Id**注解在字段上，之后Hibernate存储和加载实例的时候，会直接访问类相应的字段，如果注解在访问器方法上，那之后Hibernate会访问相应的访问器方法。

## 配置键生成器
### GenerationType
通常需要Hibernate自动生成一个代理键值，可以通过**@GeneratedValue(strategy = ...)**来指定，其中strategy是**GenerationType**类型的枚举值，其枚举值如下：

- **GenerationType.AUTO** ： Hibernate会根据hibernate.cfg.xml文件当中配置的数据库方言自动选择一种合适的策略，是默认的配置。
- **GenerationType.SEQUENCE** ： Hibernate会在数据库中创建一个名为HIBERNATE_SEQUENCE的序列，如果数据库不支持序列的话，那么Hibernate将会创建一个只有一个列的表来模拟序列的行为，该序列会在INSERT之前调用，用来生成唯一的主键。
- **GenerationType.IDENTITY** ： Hibernate会在建表的DDL语句当中生成一个自增长列，用于生成代理键。
- **GenerationType.TABLE** ： Hibernate通过创建额外的表来单独保存主键值。

### 自定义Generator
可以通过**@GenericGenerator**注解来自定义生成器，一般放在package-info.java中作为包级元注解，如下所示：

	@org.hibernate.annotations.GenericGenerator(
        name = "CUSTOM_GENERATOR",
        strategy = "enhanced-sequence",
        parameters = {
                @org.hibernate.annotations.Parameter(
                        name = "sequence_name",
                        value = "custom_sequence"
                ),
                @org.hibernate.annotations.Parameter(
                        name = "initial_value",
                        value = "1000"
                )
        }
	)
	package com.stephen.hibernatepractice;
其中strategy对应的是标识符生成器策略，名为"sequence_name"的参数代表着序列的名称（如果数据库不支持序列则代表序列表的名称），initial_value代表的是生成键的初始值，以后以此值为基础进行递增。

#### 生成器策略
对于生成器策略，如果想让Hibernate自动进行选择，那么可以开启GenerationType.AUTO，Hibernate会根据设置的数据库方言选择合适的解决方案，最有可能是序列或者是标识，但是可能不是最佳的可移植方案。如果需要灵活的可移植行为，则可以采用上述的"enhanced-sequence"。关于完整的主键生成策略，可以参见：[Hibernate主键生成策略](http://www.cnblogs.com/flyoung2008/articles/2165759.html)。

需要注意的是，对于某些生成器策略，主键id并不能立即生成，有可能是在INSERT之后才生成，因此可能存在**em.persist(something)**之后，**something.getId()**为空，因此应当尽量选择插入前生成策略的主键生成策略，一般情况下"enhanced-sequence"是最好的选择。

## 实体映射选项
### 控制名称
可以通过下列方式控制生成的表的名称，默认情况下生成的表名称为类名的大写：

	@Entity
	@Table(name = "USERS")
	public class User {
		...
	}
### 动态SQL生成
默认情况下，Hibernate会为每一个持久化单元生成CRUD语句并缓存起来，如果一个表的结构非常巨大，那么这种缓存将影响性能。对于UPDATE语句来说，由于更新的列是未知的，那Hibernate在启动时将会生成对所有列进行更新的UPDATE语句，对于没有变化的值来说，Hibernate会将其设置为旧值。

如果需要动态生成SQL语句，可以通过**@DynamicInsert**和**@DynamicUpdate**注解来标识持久化类。
---
title: Hibernate基本使用——映射集合
date: 2017-07-13 11:50:42
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
本篇主要讲集合类的映射。
<!--more-->

# 基本类型集合映射
## 选择集合的接口
Java域模型当中集合属性的声明形式如下：

	<Interface> collection = new <Implementation>();
每个JDK接口都有Hibernate所支持的一个匹配实现，Hibernate会包装已在字段声明上初始化过的集合，并在它不正确时替代它，以达到对集合元素的延迟加载和脏检查。

如果不对Hibernate进行扩展，可以从以下集合中选择：

- **java.util.Set**: 使用java.util.HashSet初始化，不保存元素的顺序，不允许重复的元素，所有的JPA都支持此类型。
- **java.util.SortedSet**: 使用java.util.TreeSet来进行初始化，这个集合支持元素的固定顺序，数据项的排列发生在内存之中，在Hibernate加载数据后进行排列，仅用于Hibernate的扩展。
- **java.util.List**: 使用java.util.ArrayList初始化。Hibernate会在表中使用额外的列保存每个元素的位置。
- **java.util.Collection**: 使用java.util.ArrayList属性进行初始化，不保存元素的顺序，但是允许重复元素。
- **java.util.Map**: 使用java.util.HashMap进行初始化，不保存元素的顺序。
- **java.util.SortedMap**: 使用java.util.TreeMap进行初始化，可以保存元素的顺序，排列在内存当中进行，在Hibernate记载数据之后进行排列。

下面通过演示来看几种基本类型集合的映射。
## 映射集
映射Set接口，要采用HashSet进行实现。

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @Column(name = "FILE_NAME")
	    protected Set<String> images = new HashSet<>();
	
	    ...
	}
	
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        FILE_NAME varchar(255)
    ) engine=MyISAM
    
其中Item\_id就是外层Item的id，这样通过Set集合就保证了无重复元素，因此IMAGES实质上的主键是Item\_id和FILE_NAME的组合键，即便Hibernate没有指定。

## 映射包
包是允许重复元素的未排序集合，采用Collection作为接口，ArrayList进行实现。

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @org.hibernate.annotations.CollectionId(columns = @Column(
	            name = "IMAGE_ID"),
	            type = @org.hibernate.annotations.Type(type = "long"),
	            generator = Constants.CUSTOM_GENERATOR)
	    @Column(name = "FILE_NAME")
	    protected Collection<String> images = new ArrayList<>();
	
	    ...
	}
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        FILE_NAME varchar(255),
        IMAGE_ID bigint not null,
        primary key (IMAGE_ID)
    ) engine=MyISAM
    
    alter table IMAGES 
       add constraint FK6yom4y18w9wytd67sme4nu2q7 
       foreign key (Item_id) 
       references Item (id)
因为允许重复的元素，因此为了满足第一范式，应该为此表创建一个代理主键，通过Hibernate的CollectionId来创建。
## 映射列表
**List**接口允许保存元素的位置，因此若要按照用户存储的顺序显示元素，可以采用这个接口并将ArrayList作为实现：

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @OrderColumn
	    @Column(name = "FILE_NAME")
	    protected List<String> images = new ArrayList<>();
	
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
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        FILE_NAME varchar(255),
        images_ORDER integer not null,
        primary key (Item_id, images_ORDER)
    ) engine=MyISAM
Item\_id和images\_ORDER作为复合主键，保证了重复元素和顺序。

需要注意的是，假如表中原来有三个元素A, B, C按照顺序存储，删除A记录，则Hibernate将为B, C执行Update语句使它们向左移动来填充间隔。
## 映射键／值对
可以通过Map接口来映射键值对，这里Map中的键值和Item的主键作为复合主键存到集合对应的表：

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @MapKeyColumn(name = "FILENAME")
	    @Column(name = "IMAGENAME")
	    protected Map<String, String> images = new HashMap<>();
	
	    ...
	}
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        IMAGENAME varchar(255),
        FILENAME varchar(255) not null,
        primary key (Item_id, FILENAME)
    ) engine=MyISAM
## 排序集合
要相对集合进行排序可以采取两种方式，一种是使用Java比较器对结果在内存中进行排列，另一种是直接利用数据库的ORDER BY子句来使结果变得有序。

### 使用SortedSet

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @Column(name = "IMAGENAME")
	    @org.hibernate.annotations.SortComparator(ReverseStringComparator.class)
	    protected SortedSet<String> images = new TreeSet<>();
	
	    ...
	}
此时从数据库当中读取数据，Hibernate加载完毕之后会在内存当中对其进行排序处理。

**注意：**此种操作不是双向的，也就是说，只能保证读取的数据是有序的，不能保证写入的数据是有序的。
### 使用ORDER BY子句
可以使用**@org.hibernate.annotations.OrderBy**注释来从数据库中得到数据的有序集合，此时排序在数据库当中执行：

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column
	    protected String name;
	
	    @ElementCollection
	    @CollectionTable(name = "IMAGES")
	    @Column(name = "IMAGENAME")
	    @org.hibernate.annotations.OrderBy(clause = "IMAGENAME desc")
	    protected List<String> images = new ArrayList<>();
	
	    ...
	}
clause为order by子句当中的后面一部分，甚至可以调用SQL函数。
# 映射组件集合
对于前面一个Item对应于多个String类型的image而言，将Image作为一个嵌入类通常是更加优雅的做法：

	@Embeddable
	public class Image {
	    @Column(nullable = false)
	    protected String title;
	
	    @Column(nullable = false)
	    protected String filename;

	    protected int width;
	
	    protected int height;
	
	    public Image(String title, String filename, int width, int height) {
	        this.title = title;
	        this.filename = filename;
	        this.width = width;
	        this.height = height;
	    }
	
	    public Image() {
	
	    }
	    
	    ...
	}
## 定义组件的相等性
若要将组件放在Map作为键的话，需要重写组件的**equals()**和**hashCode()**方法，还需要在可以唯一标识对象实例的属性上指定**@Column(nullable = false)**。如上面可以指定title和filename作为嵌入类的超键，重写两个方法：

	// 由IntelliJ自动生成
	@Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Image image = (Image) o;

        if (!title.equals(image.title)) return false;
        return filename.equals(image.filename);
    }

    @Override
    public int hashCode() {
        int result = title.hashCode();
        result = 31 * result + filename.hashCode();
        return result;
    }
值得注意的是，指定嵌入类超键时，如果超键的类型是字符串类型，那就尽量在**@Column**注解里面指定位长，否则生成主键时可能因为复合主键太长而发生错误。
## 组件集
映射组件的Set集合：

	@ElementCollection
    @CollectionTable(name = "IMAGES")
    protected Set<Image> images = new HashSet<>();
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        filename varchar(30) not null,
        height integer,
        title varchar(50) not null,
        width integer,
        primary key (Item_id, filename, title)
    ) engine=MyISAM
## 组件包
映射组件的Collection集合：

	@ElementCollection
    @CollectionTable(name = "IMAGES")
    @org.hibernate.annotations.CollectionId(columns = @Column(name = "IMAGE_ID"),
            type = @org.hibernate.annotations.Type(type = "long"),
            generator = Constants.CUSTOM_GENERATOR)
    protected Collection<Image> images = new ArrayList<>();
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        filename varchar(30) not null,
        height integer,
        title varchar(50) not null,
        width integer,
        IMAGE_ID bigint not null,
        primary key (IMAGE_ID)
    ) engine=MyISAM
此时嵌入类的超键不作为主键的一部分。
## 作为映射键的部分
要作为HashMap的键部分进行映射，则需要把嵌入类的超键作为主键：

	@Embeddable
	public class Filename {
	    @Column(nullable = false, length = 30)
	    protected String name;
	    
	    @Column(nullable = false, length = 30)
	    protected String extension;
	
	    @Override
	    public boolean equals(Object o) {
	        if (this == o) return true;
	        if (o == null || getClass() != o.getClass()) return false;
	
	        Filename filename = (Filename) o;
	
	        if (!name.equals(filename.name)) return false;
	        return extension.equals(filename.extension);
	    }
	
	    @Override
	    public int hashCode() {
	        int result = name.hashCode();
	        result = 31 * result + extension.hashCode();
	        return result;
	    }
	}
	
	@ElementCollection
    @CollectionTable(name = "IMAGES")
    protected Map<Filename, Image> images = new HashMap<>();
生成的sql语句：

	create table IMAGES (
       Item_id bigint not null,
        filename varchar(30) not null,
        height integer,
        title varchar(50) not null,
        width integer,
        extension varchar(30) not null,
        name varchar(30) not null,
        primary key (Item_id, extension, name)
    ) engine=MyISAM
作为键的嵌入类的超键为生成表的复合主键，注意要指定每个不为null的属性的长度，否则可能会发生(笔者是MySQL环境)：

	Specified key was too long; max key length is 1000 bytes
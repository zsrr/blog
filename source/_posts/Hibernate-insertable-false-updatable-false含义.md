---
title: 'Hibernate insertable = false, updatable = false含义'
date: 2017-07-17 14:06:21
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
对Hibernate的学习卡在了对insertable = false和updatable = false的理解上，故做此文章以记之。
<!--more-->

根据[官方文档](http://docs.oracle.com/javaee/5/api/javax/persistence/Column.html)来看，这两个参数决定着此列是否出现在SQL INSERT/UPDATE语句当中。

# updatable = false
这个很容易被理解，被如此设置的列只能在首次被插入，之后不能被更改。更新数据后，Hibernate不会将其写到update语句当中。
# insertable = false
让我感到困惑的地方来了，既然列不能出现在INSERT语句当中，那这个列存在的意义是什么呢？

有时数据库会自己生成值，例如生成时间戳，生成默认值以及每次修改运行触发器。

可以使用**@org.hibernate.annotations.Generated**来标记生成属性。
## 生成时间戳
例如要想保存一个日期属性，每当更新行记录时便更新它，此时可以采用生成属性：

	@Temporal(TemporalType.TIMESTAMP)
    @Column(insertable = false, updatable = false)
    @org.hibernate.annotations.Generated( GenerationTime.ALWAYS )
    protected Date lastModified;
**注意：在MySQL5下此不能正常工作，请用@UpdateTimestamp代替。**
## 默认值
要让某个属性拥有默认值，也可以使用生成属性：

	@Column(insertable = false)
    @org.hibernate.annotations.ColumnDefault("1.00")
    @org.hibernate.annotations.Generated( GenerationTime.INSERT )
    protected BigDecimal price;
## 复合主键
当复合主键中的键值是来自于其他实体的外键时，此时应该在**@JoinColumn**注解中指定insertable和updateble为false，如上一篇文章中指定Category类和Item类之间多对多关联的中间实体类：

	@Entity
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
	    }
	
	    public CategorizedItem() {
	    }
	}
主键可以在构造函数内合成，所以两个属性只能是只读的。

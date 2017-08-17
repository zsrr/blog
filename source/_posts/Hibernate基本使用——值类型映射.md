---
title: Hibernate基本使用——值类型映射
date: 2017-06-21 09:11:03
tags: [数据库,ORM,Hibernate,Java Web]
categories: ORM

---
上一篇讲了实体映射及主键生成策略，这一篇主要讲持久化类属性的映射策略。
<!--more-->

# 映射基本属性
## 基本类型映射
基本类型的映射规则如下：

- 如果属性是基元或者基元包装器，如String, BigInteger, BigDecimal, Date, Calendar, byte[], char[]，则它们会被自动持久化。
- 如果属性所属的类标记为**@Embeddable**，那么Hibernate会将其映射为多列，稍后再说。
- 如果属性的类是**java.io.Serializable**，那么Hibernate会将其值以序列化的方式存储，此时存储的是一系列字节（很蠢）。

## @Column注解
此注解会更改属性的默认设置，通常用作给属性添加**NOT NULL**约束，
或者设置映射到数据库表的列的名称，用法如下：

	@Column(name = "PRICE", nullable = false)
	protected BigDecimal price;
此时的price属性将会映射到表中名为"PRICE"的列，且试图添加一个price属性为null的记录将会得到异常。

此外要和Bean Validation中的**@NotNull**注解区分开来，此注解在运行时检查所标注的属性是否为空，并不能在SQL中为相应的列添加**NOT NULL**注解。

## 自定义属性访问
在默认情况下，如果一个持久化类的**@Id**注解位于字段上，那么之后Hibernate存储和加载类实例时会直接访问字段。如果**@Id**位于get方法上，那么Hibernate之后会通过访问器间接访问字段。

如果要修改默认设置，那么可以通过@Access注解来标记字段或访问方法。

	@Entity
	public class Item {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;

	    @Access(AccessType.PROPERTY)
	    @Column(name = "ITEM_NAME", nullable = false)
	    protected String name;
	
	    public String getName() {
	        return name;
	    }
	}
此后对name属性的更改就会通过访问器的方式，而非直接访问该字段。

## 派生属性
采用**@org.hibernate.annotations.Formula**注解来声明派生属性，派生属性在持久化类被检索的时候根据SQL语句进行估算，如下所示：

	@org.hibernate.annotations.Formula(
		"substr(DESCRIPTION, 1, 12) || '...'"
	)
	protected String shortDescription;
	
	@org.hibernate.annotations.Formula(
		"select avg(b.AMOUNT) from BID b where b.ITEM_ID = ID"
	)
	protected BigDecimal averageBidAmount;
注意的是，估算的属性是只读的，其只能出现在**select**语句当中，绝不会出现在**update**和**insert**当中。
## 转换列值
假设数据库表中有一个**YUAN**的数据库列，现在将其映射到持久化类的属性当中，我们希望能够以分的形式来呈现。当然，在数据检索出来之后手动转换是一个不错的选择，但是我们可以利用**@org.hibernate.annotations.ColumnTransformer**注解进行列转换，如下所示：

	@Column(name = "YUAN")
    @org.hibernate.annotations.ColumnTransformer(
            read = "YUAN * 10.0",
            write = "? / 10.0"
    )
    protected BigDecimal price;
## 生成默认值
有关生成默认值的博文地址如下：[hibernate之生成的和默认的属性值(使用generated刷新实体)](http://blog.csdn.net/fhd001/article/details/5878498)，不过经过笔者的实践，这个貌似在MySQL环境当中存在着问题，具体请参见：[Hibernate @Generated annotation doesn't work well](https://stackoverflow.com/questions/44593294/hibernate-generated-annotation-doesnt-work-well)
## 枚举映射
可以通过以下方式映射一个枚举属性：

	public enum Type {
        BOOK,
        CAR,
        FOOD
    }

    @NotNull
    @Enumerated(EnumType.STRING)
    protected Type type;
type会以字符串的形式存到数据库表当中。
# 映射可嵌入组件
可以用**@Embeddable**注解声明一个类是可嵌入的，可嵌入的类的每个属性将会被映射到外层持久化类所映射的表的单个列当中。举个例子，想要在USER表当中存储关于用户的信息，其中有其地址信息，地址可以作为一个可嵌入类来定义：

	@Embeddable
	public class Address {
	    @Column(nullable = false)
	    protected String city;
	
	    @Column(nullable = false)
	    protected String zipCode;
	
	    @Column(nullable = false)
	    protected String street;
	
	    public Address() {
	    }
	
	    public Address(String city, String zipCode, String street) {
	        this.city = city;
	        this.zipCode = zipCode;
	        this.street = street;
	    }
	
	    public String getCity() {
	        return city;
	    }
	
	    public void setCity(String city) {
	        this.city = city;
	    }
	
	    public String getZipCode() {
	        return zipCode;
	    }
	
	    public void setZipCode(String zipCode) {
	        this.zipCode = zipCode;
	    }
	
	    public String getStreet() {
	        return street;
	    }
	
	    public void setStreet(String street) {
	        this.street = street;
	    }
	}
User定义：

	@Entity
	public class User {
	    @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	    protected Long id;
	
	    @Column(nullable = false)
	    String name;
	
	    protected Address address;
	
	    public String getName() {
	        return name;
	    }
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public Address getAddress() {
	        return address;
	    }
	
	    public void setAddress(Address address) {
	        this.address = address;
	    }
	
	    public Long getId() {
	        return id;
	    }
	}
此时USER表的结构是这样的：

	SHOW CREATE TABLE USER;
	
	CREATE TABLE `USER` (
	  `id` bigint(20) NOT NULL,
	  `city` varchar(255) NOT NULL,
	  `street` varchar(255) NOT NULL,
	  `zipCode` varchar(255) NOT NULL,
	  `name` varchar(255) NOT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=MyISAM DEFAULT CHARSET=gbk
## 重写嵌入属性
可以通过以下方式重写嵌入的属性：

	@AttributeOverrides({
            @AttributeOverride(name = "street", column = @Column(name = "USER_STREET")),
            @AttributeOverride(name = "zipCode", column = @Column(name = "USER_ZIP_CODE")),
            @AttributeOverride(name = "city", column = @Column(name = "USER_CITY"))
    })
    protected Address address;
**@AttributeOverride**当中name对应的是Address的属性名，column对应的是要重新映射的列的信息。
# 转换器
我们已经知道在映射**java.lang.String**等基本类型的时候，Hibernate会将其映射到SQL基本类型，除了基本内置类型之外，可以定义转换器和用户类型。

## 内置类型
常见的内置类型如下：

|名称|Java类型|ANSISQL类型|
|---|--------|----------|
|integer|java.lang.Integer|INTEGER|
|long|java.lang.Long|BIGINT|
|float|java.lang.Float|FLOAT|
|double|java.lang.Double|DOUBLE|
|boolean|java.lang.Boolean|BOOLEAN|
|big_decimal|java.lang.BigDecimal|NUMERIC|
|big_integer|java.lang.BigInteger|NUMERIC|
|String|java.lang.String|VARCHAR|
|yes_no|java.lang.Boolean|CHAR(1), 'Y' or 'N'|
|class|java.lang.Class|VARCHAR|
|currency|java.util.Currency|VARCHAR|
|date|java.util.Date,java.sql.Date|DATE|
|time|java.util.Date,java.sql.Time|TIME|
|timestamp|java.util.Date,java.sql.Timestamp|TIMESTAMP|
|calendar|java.util.Calendar|TIMESTAMP|

名称为Hibernate特有，用于自定义类型映射，如：

	@org.hibernate.annotations.Type(type = "yes_no")
	protected boolean verified;
现在VERIFIED所对应的SQL类型应该为CHAR(1)，且其值为'Y'或'N'。
## JPA转换器
可以使用JPA原生转换器来完成自定义类型到另一个基本类型的转换，如下所示：

MonetaryAmount类：

	public class MonetaryAmount implements Serializable {
	    static final long serialVersionUID = 1L;
	
	    protected final BigDecimal value;
	    protected final Currency currency;
	
	    public MonetaryAmount(BigDecimal value, Currency currency) {
	        this.value = value;
	        this.currency = currency;
	    }
	
	    public BigDecimal getValue() {
	        return value;
	    }
	
	    public Currency getCurrency() {
	        return currency;
	    }
	
	    @Override
	    public boolean equals(Object o) {
	        if (this == o) return true;
	        if (o == null || getClass() != o.getClass()) return false;
	
	        MonetaryAmount that = (MonetaryAmount) o;
	
	        if (!value.equals(that.value)) return false;
	        return currency.equals(that.currency);
	    }
	
	    @Override
	    public int hashCode() {
	        int result = value.hashCode();
	        result = 31 * result + currency.hashCode();
	        return result;
	    }
	
	    @Override
	    public String toString() {
	        return value + " " + currency;
	    }
	
	    public static MonetaryAmount fromString(String str) {
	        String[] split = str.split(" ");
	        return new MonetaryAmount(new BigDecimal(split[0]), Currency.getInstance(split[1]));
	    }
	}
	
MonetaryAmountConverter:

	@Converter(autoApply = true)
	public class MonetaryAmountConverter implements AttributeConverter<MonetaryAmount, String> {
	
	    @Override
	    public String convertToDatabaseColumn(MonetaryAmount monetaryAmount) {
	        return monetaryAmount.toString();
	    }
	
	    @Override
	    public MonetaryAmount convertToEntityAttribute(String s) {
	        return MonetaryAmount.fromString(s);
	    }
	}
	
则在持久化类里面可以这样定义：

	@NotNull
	//可选，因为设置了autoApply
    @Convert(converter = MonetaryAmountConverter.class)
    @Column(name = "PRICE")
    protected MonetaryAmount ma;
此时Hibernate便会先将此属性转成相应的字符串再将其存到数据库中。

## UserType扩展
JPA转换器存在诸多限制。其不支持单值到多个列的转换，另一个限制是与查询引擎的集成，不能够在HQL查询语句当中直接访问被转换类型的属性，如：
**"select i from Item i where i.ma.value > 100"**。如果需要更加灵活的扩展，那就需要Hibernate提供的扩展接口。

可以在org.hibernate.usertype包下找到合适的扩展接口，常见接口如下：

- **UserType**——可以通过与底层JDBC的PreparedStatement和ResultSet交互来转换值。
- **CompositeUserType**——扩展了上述的**UserType**接口，提供了更多与适配类的相关的信息。例如，可以告知MonetaryAmount有两个属性：value, currency，然后就可以在select查询语句当中引用这些属性。
- **ParameterizedUserType**——提供对适配器参数的设置。
- **DynamicParameterizedUserType**——允许访问适配器的动态信息，如映射列和表名称，可以替代**ParameterizedUserType**。

### 定义用户类型
如下定义了**MonetaryAmount**类的UserType，如下所示：

	public class MonetaryAmountType implements CompositeUserType {
	    
	    // 获取类内部属性的名字
	    @Override
	    public String[] getPropertyNames() {
	        System.out.println("getPropertyNames() called");
	        return new String[]{ "value", "currency" };
	    }
	    
	    // 得到属性的类型
	    @Override
	    public Type[] getPropertyTypes() {
	        System.out.println("getPropertyTypes() called");
	        return new Type[]{ StandardBasicTypes.BIG_DECIMAL, StandardBasicTypes.CURRENCY };
	    }
	    
	    // 获取指定位置上属性的值
	    @Override
	    public Object getPropertyValue(Object o, int i) throws HibernateException {
	        System.out.println("getPropertyValue() called, and i is " + i);
	        MonetaryAmount ma = (MonetaryAmount) o;
	        if (i == 0)
	            return ma.getValue();
	        return ma.getCurrency().toString();
	    }
	
	    @Override
	    public void setPropertyValue(Object o, int i, Object o1) throws HibernateException {
	        throw new UnsupportedOperationException("setProperty() cannot be called");
	    }
	    
	    // 返回要转化的类型
	    @Override
	    public Class returnedClass() {
	        System.out.println("returnedClass() called");
	        return MonetaryAmount.class;
	    }
	    
	    // 判断两对象是否相同
	    @Override
	    public boolean equals(Object o, Object o1) throws HibernateException {
	        System.out.println("equals() called");
	        return o.equals(o1);
	    }
	
	    @Override
	    public int hashCode(Object o) throws HibernateException {
	        System.out.println("hashCode() called");
	        return o.hashCode();
	    }
	    
	    // 查询的结果集到相应类的转变
	    @Override
	    public Object nullSafeGet(ResultSet resultSet, String[] strings, SharedSessionContractImplementor sharedSessionContractImplementor, Object o) throws HibernateException, SQLException {
	        System.out.println("nullSafeGet() called");
	        if (resultSet.wasNull())
	            return null;
	        BigDecimal value = resultSet.getBigDecimal(strings[0]);
	        Currency currency = Currency.getInstance(resultSet.getString(strings[1]));
	        return new MonetaryAmount(value, currency);
	    }
	
	    @Override
	    public void nullSafeSet(PreparedStatement preparedStatement, Object o, int i, SharedSessionContractImplementor sharedSessionContractImplementor) throws HibernateException, SQLException {
	        System.out.println("nullSafeSet() called and i is " + i);
	        if (o == null) {
	            preparedStatement.setNull(i, StandardBasicTypes.BIG_DECIMAL.sqlType());
	            preparedStatement.setNull(i + 1, StandardBasicTypes.CURRENCY.sqlType());
	        } else {
	            MonetaryAmount ma = (MonetaryAmount) o;
	            preparedStatement.setBigDecimal(i, ma.getValue());
	            preparedStatement.setString(i + 1, ma.getCurrency().toString());
	        }
	    }
	    
	    // 用于获取值的副本，不可变对象返回本身即可
	    @Override
	    public Object deepCopy(Object o) throws HibernateException {
	        return o;
	    }
	    
	    // 如果对象是不可变的，Hibernate会采取优化措施
	    @Override
	    public boolean isMutable() {
	        return false;
	    }
	    
	    // 返回序列化表示的值
	    @Override
	    public Serializable disassemble(Object o, SharedSessionContractImplementor sharedSessionContractImplementor) throws HibernateException {
	        System.out.println("disassemble() called");
	        return o.toString();
	    }
	    
	    // 用序列化的值创造对象
	    @Override
	    public Object assemble(Serializable serializable, SharedSessionContractImplementor sharedSessionContractImplementor, Object o) throws HibernateException {
	        System.out.println("assemble() called");
	        return MonetaryAmount.fromString((String) serializable);
	    }
	    
	    // 在EntityManager#merge()操作期间调用，需要返回原始对象的副本
	    @Override
	    public Object replace(Object o, Object o1, SharedSessionContractImplementor sharedSessionContractImplementor, Object o2) throws HibernateException {
	        return o;
	    }
	}
以上代码有注释，不再细说。
### 使用用户类型
使用用户类型时，可以先用包级注释声明类型：

	@org.hibernate.annotations.TypeDefs(
        @org.hibernate.annotations.TypeDef(name = "monetary_amount_type",
                typeClass = com.stephen.hibernatepractice.converter.MonetaryAmountType.class)
	)
	package com.stephen.hibernatepractice;
在实体类中引用：

	@NotNull
    @org.hibernate.annotations.Type(type = "monetary_amount_type")
    @org.hibernate.annotations.Columns(columns = {
            @Column(name = "PRICE"),
            @Column(name = "CURRENCY", length = 3)
    }
    )
    protected MonetaryAmount monetaryAmount;
则monetaryAmount属性的value映射为PRICE列，currency映射为CURRENCY列，并且此时可以在HQL select语句里面引用monetaryAmount属性的属性。
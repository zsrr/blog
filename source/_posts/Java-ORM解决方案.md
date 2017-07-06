---
title: Java ORM解决方案
date: 2017-04-14 15:31:14
tags: Java
categories: Java

---
**注：**此篇部分内容，图片来自[JPA Tutorial](http://ok34fi9ya.bkt.clouddn.com/jpa_tutorial.pdf)

# 什么是ORM
对象关系映射(Object Relational Mapping)，用面向对象的语言实现了数据的转换，存储等操作。
# Java ORM解决方案
## Java Persistence Api
Java持久化API（以下简称JPA），是从EJB发展而来的Java ORM框架。
### 主要结构
其主要结构如下图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%887.11.50.png)
主要组成部分有下面几个：

- **POJO class**此种类定义了像id, name等等基本属性，相当于关系数据库的表。
- **Service class**此种类实现了与数据库进行操作，包括获取对象，删除对象，更新对象等操作。
- **JPA Provider**由各大厂商提供的对JPA的具体实现，如Eclipselink, Toplink, Hibernate等等。
- **Mapping file**用来定义POJO class和关系数据库的映射关系。

除了Mapping file，JPA还提供了各种形式的注解来定义POJO class和关系数据库之间的映射关系，主要的几个注解如下：

- **@Entity**用来声明一个将要和关系数据库发生映射的POJO class。
- **@Table**用来声明此类对应的表名, schema和catalog。
- **@Basic**通常用来声明属性的加载方式。
- **@Id**用来声明POJO class对应的表的主键。
- **@GeneratedValue**用来指定主键的生成策略。
- **@Column**用来声明属性对应的列。

此外，POJO class应该具备Java Bean的标准形式。

### 用IntelliJ IDEA建立JPA项目
笔者采用的环境是IDEA 2017.1.1，Hibernate 5.2.9，JPA 2.1。注意Hibernate的版本要和JPA的版本对应，不然会出错。
#### 新建项目
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%887.37.05.png)
勾选JavaEE Persistence项目，下面指定JPA的版本为2.1，Provider为Hibernate，建好项目后整个项目的结构如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%887.42.46.png)
#### 加入Hibernate以及数据库驱动依赖
笔者采用的是h2数据库，将相应的依赖包放在lib目录下，对用的jar依赖包的下载地址：[h2以及Hibernate依赖包](http://ok34fi9ya.bkt.clouddn.com/jpa-with-hibernate.zip)

将jar包放在lib文件夹之后，打开Project Struture，找到项目对应的Module，为其添加dependicies:
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%887.59.23.png)
#### 配置persistence.xml文件
此文件定义了数据源，映射类等基本信息，且一定要在项目类路径下的META-INF文件夹下，笔者采用的是h2数据库，配置如下：

	<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">

	    <persistence-unit name="HibernatePersistenceUnit">
	        <!--定义Provider-->
	        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
	        <properties>
	            <property name="hibernate.connection.url" value="jdbc:h2:~/test/test"/>
	            <property name="hibernate.connection.driver_class" value="org.h2.Driver"/>
	            <property name="hibernate.connection.username" value="sa"/>
	            <!--<property name="hibernate.connection.password" value=""/>-->
	            <property name="hibernate.hbm2ddl.auto" value="create"/>
	        </properties>
	    </persistence-unit>
	</persistence>
注意，笔者采用的是h2嵌入式数据库，因此不用提供用户的密码，还要注意里面的hbm2ddl.auto属性，此属性的值在这里是create，意味着应用程序每次启动都会创建一个新的表格，而上一次启动创建的表格将不复存在，关于此属性更多的内容，请见：[hibernate.hbm2ddl.auto属性配置](http://blog.csdn.net/kjfcpua/article/details/4272415)

至此，项目的基本配置完毕。
### 基本使用
此小节通过小小的例子来阐述JPA的基本使用。

Employee.java:

	@Entity
	@Table(name = "Employee", schema = "PUBLIC", catalog = "TEST")
	public class Employee {
	    @Id
	    private int eid;
	    private String ename;
	    private double salary;
	    private String deg;
	
	    public Employee(String ename, double salary, String deg) {
	        this.ename = ename;
	        this.salary = salary;
	        this.deg = deg;
	    }
	
	    public Employee() {
	
	    }
	
	    public int getEid() {
	        return eid;
	    }
	
	    public void setEid(int eid) {
	        this.eid = eid;
	    }
	
	    public String getEname() {
	        return ename;
	    }
	
	    public void setEname(String ename) {
	        this.ename = ename;
	    }
	
	    public double getSalary() {
	        return salary;
	    }
	
	    public void setSalary(double salary) {
	        this.salary = salary;
	    }
	
	    public String getDeg() {
	        return deg;
	    }
	
	    public void setDeg(String deg) {
	        this.deg = deg;
	    }
	    
		@Override
	    public String toString() {
	        return "Employee{" +
	                "eid=" + eid +
	                ", ename='" + ename + '\'' +
	                ", salary=" + salary +
	                ", deg='" + deg + '\'' +
	                '}';
	    }
	}
Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	    private static void createAnEmployee(Employee employee) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	        manager.persist(employee);
	        transaction.commit();
	        manager.close();
	    }
	
	    private static Employee findEmployee(int eid) {
	        EntityManager manager = factory.createEntityManager();
	        try {
	            return manager.find(Employee.class, eid);
	        } finally {
	            manager.close();
	        }
	    }
	
	    private static void deleteEmployee(int eid) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	        Employee employee = manager.find(Employee.class, eid);
	        manager.remove(employee);
	        transaction.commit();
	        manager.close();
	    }
	
	    private static void updateEmployeeSalary(int eid, double salary) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	        Employee employee = manager.find(Employee.class, eid);
	        System.out.println(employee);
	        employee.setSalary(salary);
	        transaction.commit();
	        manager.close();
	    }
	
	
	    public static void main(String[] args) {
	        Employee employee = new Employee();
	
	        employee.setEid(100);
	        employee.setEname("zsr");
	        employee.setSalary(3000);
	        employee.setDeg("Student");
	
	        createAnEmployee(employee);
	        System.out.println(findEmployee(100));
	        updateEmployeeSalary(100, 5000);
	        System.out.println(findEmployee(100));
	        deleteEmployee(100);
	        factory.close();
	    }
	}
结果如下：

	Employee{eid=100, ename='zsr', salary=3000.0, deg='Student'}
	Employee{eid=100, ename='zsr', salary=3000.0, deg='Student'}
	Employee{eid=100, ename='zsr', salary=5000.0, deg='Student'}
	
需要注意的是：如果要手动指定主键，就不要用**@GeneratedValue**去注释主键，否则会出现detached entity passed to persist异常。

以上主要用到的类有下面几个：

- **EntityManagerFactory**  EntityManager的工厂类，与EntityManager是一对多的关系。
- **EntityManager** 用来执行对象持久化的类，是**Query**类的工厂类。
- **EntityTransaction** 与EntityManager是一对一的关系，作用在EntityManager上更新表的操作存储在其中，最后用其commit()方法才正式生效。
- **Persistence** 提供静态方法用来生成EntityManagerFactory对象。

关系如下图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%889.35.54.png)
### JPQL
JPQL是在SQL语句基础之上建立起来的标准语句，能够执行对表中对象的**SELECT, UPDATE, DELETE**。
#### 基本使用
main方法改成如下代码：

	@SuppressWarnings("unchecked")
    public static void main(String[] args) {
        Employee employee1 = new Employee(100, "Gopal", 40000, "Technical Manager");
        Employee employee2 = new Employee(101, "Manisha", 35000, "Reader");
        Employee employee3 = new Employee(102, "Masthanvali", 37000, "Teacher");
        Employee employee4 = new Employee(103, "Satish", 30000, "Student");
        Employee employee5 = new Employee(104, "Krishna", 31000, "Tutor");
        Employee employee6 = new Employee(105, "Kiran", 32000, "Drinker");

        createAnEmployee(employee1);
        createAnEmployee(employee2);
        createAnEmployee(employee3);
        createAnEmployee(employee4);
        createAnEmployee(employee5);
        createAnEmployee(employee6);

        EntityManager manager = factory.createEntityManager();

        Query betweenQuery = manager.createQuery("select e from Employee e where e.salary between 30000 and 35000");
        List<Employee> salaryQueryEmployees = (List<Employee>) betweenQuery.getResultList();

        for (Employee employee : salaryQueryEmployees) {
            System.out.println(employee);
        }

        System.out.println();

        Query likeQuery = manager.createQuery("select  e from Employee e where e.ename like 'M%'");
        List<Employee> likeQueryResults = (List<Employee>) likeQuery.getResultList();

        for (Employee employee : likeQueryResults) {
            System.out.println(employee);
        }

        manager.close();
        factory.close();
    }
运行结果：

	Employee{eid=101, ename='Manisha', salary=35000.0, deg='Reader'}
	Employee{eid=103, ename='Satish', salary=30000.0, deg='Student'}
	Employee{eid=104, ename='Krishna', salary=31000.0, deg='Tutor'}
	Employee{eid=105, ename='Kiran', salary=32000.0, deg='Drinker'}
	
	Employee{eid=101, ename='Manisha', salary=35000.0, deg='Reader'}
	Employee{eid=102, ename='Masthanvali', salary=37000.0, deg='Teacher'}
	
sql语法实在是繁多，这里不再一一详细列举。
#### NamedQuery
**@NamedQuery**注解用实现定义好的且不可变的query语句来执行查询操作，这个query语句中含有未知的参数可以在别处指定，用法如下：

Employee.java:

	@Entity
	@Table(name = "Employee", schema = "PUBLIC", catalog = "TEST")
	@NamedQuery(query = "select e from Employee e where e.eid = :id", name = "find employee by id")
	public class Employee {
		...
	}
	
main函数：

	@SuppressWarnings("unchecked")
    public static void main(String[] args) {
        ...

        EntityManager manager = factory.createEntityManager();

        Query query = manager.createNamedQuery("find employee by id");
        query.setParameter("id", 103);

        Employee employee = (Employee) query.getSingleResult();
        System.out.println(employee);

        manager.close();
        factory.close();
    }
### JPA对继承的处理
Java对象之间存在错综复杂的继承关系，而关系数据库没有继承这种概念，JPA对这种情况提供了多种解决方案。

考虑如下的继承关系：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%8810.25.29.png)

#### 单一表策略
单一表策略是指将不同的对象（存在继承关系）放在一个表当中，代码如下：

Staff.java:

	@Entity
	@Table(name = "Staff", schema = "PUBLIC", catalog = "TEST")
	@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
	@DiscriminatorColumn(name = "type")
	public class Staff {
	    @Id
	    private int sid;
	    private String sname;
	
	    public Staff(int sid, String sname) {
	        this.sid = sid;
	        this.sname = sname;
	    }
	
	    public Staff() {
	
	    }
	
	    public int getSid() {
	        return sid;
	    }
	
	    public void setSid(int sid) {
	        this.sid = sid;
	    }
	
	    public String getSname() {
	        return sname;
	    }
	
	    public void setSname(String sname) {
	        this.sname = sname;
	    }
	}

TeachingStaff.java:

	@Entity
	@DiscriminatorValue(value = "TS")
	public class TeachingStaff extends Staff {
	    private String qualification;
	    private String subjectexpertise;
	
	    public TeachingStaff(int sid, String sname, String qualification, String subjectexpertise) {
	        super(sid, sname);
	        this.qualification = qualification;
	        this.subjectexpertise = subjectexpertise;
	    }
	
	    public TeachingStaff() {
	        super();
	    }
	
	    public String getQualification() {
	        return qualification;
	    }
	
	    public void setQualification(String qualification) {
	        this.qualification = qualification;
	    }
	
	    public String getSubjectexpertise() {
	        return subjectexpertise;
	    }
	
	    public void setSubjectexpertise(String subjectexpertise) {
	        this.subjectexpertise = subjectexpertise;
	    }
	}
NonTeachingStaff.java:

	@Entity
	@DiscriminatorValue(value = "NST")
	public class NonTeachingStaff extends Staff {
	    private String areaexpertise;
	
	    public NonTeachingStaff(int sid, String sname, String areaexpertise) {
	        super(sid, sname);
	        this.areaexpertise = areaexpertise;
	    }
	
	    public NonTeachingStaff() {
	        super();
	    }
	
	    public String getAreaexpertise() {
	        return areaexpertise;
	    }
	
	    public void setAreaexpertise(String areaexpertise) {
	        this.areaexpertise = areaexpertise;
	    }
	}

Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	
	    @SuppressWarnings("unchecked")
	    public static void main(String[] args) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	
	        TeachingStaff ts1 = new TeachingStaff(100, "zsr", "fine", "a");
	        TeachingStaff ts2 = new TeachingStaff(101, "zx", "good", "a");
	
	        NonTeachingStaff nts1 = new NonTeachingStaff(102, "zy", "a");
	        NonTeachingStaff nts2 = new NonTeachingStaff(103, "zu", "b");
	
	        manager.persist(ts1);
	        manager.persist(ts2);
	        manager.persist(nts1);
	        manager.persist(nts2);
	        
	        transaction.commit();
	
	        manager.close();
	        factory.close();
	    }
	}
运行结束后打开h2 console，执行:

	SELECT * FROM STAFF;
运行结果：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%8810.51.27.png)
#### 合并表策略
合并表策略（不知道怎么翻译才是对的，原文是Joined Table Strategy）是指将几个对象模型中共有的东西抽出来，在此基础上为子类分别建表，代码如下：

Staff.java:

	@Entity
	@Table(name = "Staff", schema = "PUBLIC", catalog = "TEST")
	@Inheritance(strategy = InheritanceType.JOINED)
	public class Staff {
		...
	}
TeachingStaff.java:

	@Entity
	@PrimaryKeyJoinColumn(referencedColumnName = "sid")
	public class TeachingStaff extends Staff {
		...
	}
NonTeachingStaff的变化与TeachingStaff的相同，不再展示。

main方法不变，执行完毕之后，打开h2 console，运行如下指令：

	SELECT * FROM STAFF;
	SELECT * FROM TEACHINGSTAFF;
	SELECT * FROM NONTEACHINGSTAFF;
结果如下：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%8811.12.43.png)
#### 一类一表策略
顾名思义就是为每一个类都产生一个表，此方法不再做展示。
### 实体间关系
除了继承，实体之间也有很多复杂的组合关系，JPA为实体之间的关系提供了四种注解：**@OneToOne @OneToMany @ManyToOne @ManyToMany**
#### ManyToOne
考虑下面的模型：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%8811.28.37.png)
此时多个Employee可以对应一个Department，在关系数据库里面，Employee的did便是一个外键，且指向Department的id。

代码如下：

Employee.java:

	@Entity
	@Table(name = "Employee", schema = "PUBLIC", catalog = "TEST")
	public class Employee {
	    @Id
	    private int eid;
	    private String ename;
	    private double salary;
	    private String deg;
	    @ManyToOne
	    private Department department;
	    
	    ...

	}
Department.java:

	@Entity
	public class Department {
	    @Id
	    private int did;
	    private String dname;
	    
	    public Department(int did, String dname) {
	        this.did = did;
	        this.dname = dname;
	    }
	
	    public Department() {
	    }
	
	    public int getDid() {
	        return did;
	    }
	
	    public void setDid(int did) {
	        this.did = did;
	    }
	
	    public String getDname() {
	        return dname;
	    }
	
	    public void setDname(String dname) {
	        this.dname = dname;
	    }
	}
Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	
	    @SuppressWarnings("unchecked")
	    public static void main(String[] args) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	
	        Department department = new Department(10, "DingXiang");
	        manager.persist(department);
	
	        Employee employee1 = new Employee(100, "zsr", 4000, "Student");
	        employee1.setDepartment(department);
	
	        Employee employee2 = new Employee(101, "yui", 4000, "Student");
	        employee2.setDepartment(department);
	
	        Employee employee3 = new Employee(102, "yuo", 4000, "Student");
	        employee3.setDepartment(department);
	
	        manager.persist(employee1);
	        manager.persist(employee2);
	        manager.persist(employee3);
	
	        transaction.commit();
	
	        manager.close();
	        factory.close();
	    }
	} 
	
执行结束后，打开h2 console，查看Employee和Department表，结果如下：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-15%20%E4%B8%8B%E5%8D%8811.47.43.png)
#### OneToMany
上面的反过来说，一个Department可以对应多个Employee，代码修改如下：

Department.java:

	@Entity
	public class Department {
	    @Id
	    private int did;
	    private String dname;
	    @OneToMany(targetEntity = Employee.class)
	    private List<Employee> employeeList;
	
	    public Department(int did, String dname) {
	        this.did = did;
	        this.dname = dname;
	    }
	
	    public Department() {
	    }
	
	    public List<Employee> getEmployeeList() {
	        return employeeList;
	    }
	
	    public void setEmployeeList(List<Employee> employeeList) {
	        this.employeeList = employeeList;
	    }
	
	    public int getDid() {
	        return did;
	    }
	
	    public void setDid(int did) {
	        this.did = did;
	    }
	
	    public String getDname() {
	        return dname;
	    }
	
	    public void setDname(String dname) {
	        this.dname = dname;
	    }
	}
Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	
	    public static void main(String[] args) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	
	        Department department = new Department(10, "DingXiang");
	
	        Employee employee1 = new Employee(100, "zsr", 4000, "Student");
	        Employee employee2 = new Employee(101, "yui", 4000, "Student");
	        Employee employee3 = new Employee(102, "yuo", 4000, "Student");
	
	        manager.persist(employee1);
	        manager.persist(employee2);
	        manager.persist(employee3);
	
	        List<Employee> employees = new ArrayList<>();
	        employees.add(employee1);
	        employees.add(employee2);
	        employees.add(employee3);
	        
	        department.setEmployeeList(employees);
	
	        manager.persist(department);
	
	        transaction.commit();
	
	        manager.close();
	        factory.close();
	    }
	}
执行完毕之后，打开h2 console，查看表信息，如下所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-16%20%E4%B8%8A%E5%8D%8812.00.54.png)
#### OneToOne
假设上述Employee类和Department类是一一对应的关系，只需要把ManyToOne小节下Employee类中对department成员上的注解修改成**@OneToOne**，再在main方法中为每个Employee提供一个Department即可，执行完毕后，生成的表的格式和ManyToOne的格式相同，这里不再展示。
#### ManyToMany
考虑下面的模型：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-16%20%E4%B8%8A%E5%8D%8812.13.46.png)
一个课程可以有多个老师同时教学，一个老师也可以教多个课程，代码如下：

Clazz.java:

	@Entity
	@Table(name = "Clazz", schema = "PUBLIC", catalog = "TEST")
	public class Clazz {
	    @Id
	    private int cid;
	    private String cname;
	
	    @ManyToMany(targetEntity = Teacher.class)
	    private Set<Teacher> teachers;
	
	    public Clazz() {
	    }
	
	    public Clazz(int cid, String cname) {
	        this.cid = cid;
	        this.cname = cname;
	    }
	
	    public Set<Teacher> getTeachers() {
	        return teachers;
	    }
	
	    public void setTeachers(Set<Teacher> teachers) {
	        this.teachers = teachers;
	    }
	
	    public int getCid() {
	        return cid;
	    }
	
	    public void setCid(int cid) {
	        this.cid = cid;
	    }
	
	    public String getCname() {
	        return cname;
	    }
	
	    public void setCname(String cname) {
	        this.cname = cname;
	    }
	}
Teacher.java:

	@Entity
	@Table(name = "Teacher", schema = "PUBLIC", catalog = "TEST")
	public class Teacher {
	    @Id
	    private int tid;
	    private String tname;
	
	    @ManyToMany(targetEntity = Clazz.class)
	    Set<Clazz> clazzes;
	
	    public Teacher(int tid, String tname) {
	        this.tid = tid;
	        this.tname = tname;
	    }
	
	    public Teacher() {
	    }
	
	    public int getTid() {
	        return tid;
	    }
	
	    public void setTid(int tid) {
	        this.tid = tid;
	    }
	
	    public String getTname() {
	        return tname;
	    }
	
	    public void setTname(String tname) {
	        this.tname = tname;
	    }
	
	    public Set<Clazz> getClazzes() {
	        return clazzes;
	    }
	
	    public void setClazzes(Set<Clazz> clazzes) {
	        this.clazzes = clazzes;
	    }
	}
Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	
	    public static void main(String[] args) {
	        EntityManager manager = factory.createEntityManager();
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	
	        Clazz clazz = new Clazz(10, "Chinese");
	        Clazz clazz1 = new Clazz(11, "Math");
	        Clazz clazz2 = new Clazz(12, "English");
	
	        manager.persist(clazz);
	        manager.persist(clazz1);
	        manager.persist(clazz2);
	
	        Set<Clazz> clazzes = new HashSet<>();
	        clazzes.add(clazz);
	        clazzes.add(clazz1);
	        clazzes.add(clazz2);
	
	        Set<Clazz> clazzes1 = new HashSet<>(clazzes);
	        Set<Clazz> clazzes2 = new HashSet<>(clazzes);
	
	        Teacher teacher1 = new Teacher(1, "zsr");
	        Teacher teacher2 = new Teacher(2, "zx");
	        Teacher teacher3 = new Teacher(3, "zo");
	
	        teacher1.setClazzes(clazzes);
	        teacher2.setClazzes(clazzes1);
	        teacher3.setClazzes(clazzes2);
	
	        manager.persist(teacher1);
	        manager.persist(teacher2);
	        manager.persist(teacher3);
	
	        transaction.commit();
	
	        manager.close();
	        factory.close();
	    }
	}
执行完毕之后打开h2 console查看表，结果如下：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-16%20%E4%B8%8A%E5%8D%8812.33.02.png)
### Criteria Api
Criteria Api是用来执行标准查询的api，其用法遵循以下流程：

	EntityManager em = ...;	CriteriaBuilder cb = em.getCriteriaBuilder(); CriteriaQuery<Entity class> cq = cb.createQuery(Entity.class); 
	Root<Entity> from = cq.from(Entity.class);	cq.select(Entity);	TypedQuery<Entity> q = em.createQuery(cq);	List<Entity> allitems = q.getResultList();
还是拿JPQL小节的数据当例子，代码如下：

Main.java:

	public class Main {
	    private static final EntityManagerFactory factory = Persistence.createEntityManagerFactory("HibernatePersistenceUnit");
	
	    private static void createAnEmployee(EntityManager manager, Employee employee) {
	        EntityTransaction transaction = manager.getTransaction();
	        transaction.begin();
	        manager.persist(employee);
	        transaction.commit();
	    }
	
	
	    public static void main(String[] args) {
	        EntityManager manager = factory.createEntityManager();
	
	
	        Employee employee1 = new Employee(100, "Gopal", 40000, "Technical Manager");
	        Employee employee2 = new Employee(101, "Manisha", 35000, "Reader");
	        Employee employee3 = new Employee(102, "Masthanvali", 37000, "Teacher");
	        Employee employee4 = new Employee(103, "Satish", 30000, "Student");
	        Employee employee5 = new Employee(104, "Krishna", 31000, "Tutor");
	        Employee employee6 = new Employee(105, "Kiran", 32000, "Drinker");
	
	        createAnEmployee(manager, employee1);
	        createAnEmployee(manager, employee2);
	        createAnEmployee(manager, employee3);
	        createAnEmployee(manager, employee4);
	        createAnEmployee(manager, employee5);
	        createAnEmployee(manager, employee6);
	
	        CriteriaBuilder criteriaBuilder = manager.getCriteriaBuilder();
	        CriteriaQuery<Employee> employeeCriteriaQuery = criteriaBuilder.createQuery(Employee.class);
	        Root<Employee> from = employeeCriteriaQuery.from(Employee.class);
	
	        CriteriaQuery<Employee> betweenSelect = employeeCriteriaQuery.select(from)
	                .where(criteriaBuilder.between(from.get("salary"), 30000, 35000));
	        TypedQuery<Employee> betweenSelectTypedQuery = manager.createQuery(betweenSelect);
	        List<Employee> betweenSelectResults = betweenSelectTypedQuery.getResultList();
	
	        for (Employee employee : betweenSelectResults) {
	            System.out.println(employee);
	        }
	
	        manager.close();
	        factory.close();
	    }
	}
输出和JPQL小节的输出效果相同，可以看出Criteria更加面向对象。
## Hibernate
上面再配置JPA项目的时候我们用Hibernate作为Provider。Hibernate是一个优秀的开源Java ORM框架，既可以适配JPA也可以拿出来单独使用，有了上面的基础其使用也就变的简单自然。Hibernate还提供了缓存和检索服务，拥有拦截功能，其具体使用请查看官方文档，教程推荐: [Hibernate教程](http://wiki.jikexueyuan.com/project/hibernate/)

另外，IntelliJ IDEA也可以直接建立Hibernate项目省去了配置的麻烦...
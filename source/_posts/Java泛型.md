---
title: Java泛型
date: 2017-11-03 22:52:02
tags: [Java,泛型]
categories: Java

---
本篇记录Java泛型的学习笔记，用作记录。

<!--more-->

# 泛型使用的小技巧
不打算介绍其基本使用，直接介绍几个实用的技能。
## 使用限定符来限定类型
有时候可以通过一些限定符来限定泛型的类型，比如说有如下代码：

    // 代码来自 《Java核心技术 卷一》
    public static <T> T min(T...args) {
        T smallest = args[0];
        for (int i = 1; i < args.length; i++) {
            if (smallest.compareTo(args[i]) > 0) smallest = args[i];
        }
        return smallest;
    }
这个时候存在的问题是compareTo方法只有实现了Comparable接口的类才能使用，此时把代码改成如下形式即可正常运行：

    public static <T extends Comparable> T min(T...args) {
        T smallest = args[0];
        for (int i = 1; i < args.length; i++) {
            if (smallest.compareTo(args[i]) > 0) smallest = args[i];
        }
        return smallest;
    }

上面的``T extends Comparable``就是限定了泛型必须实现Comparable接口，可以有多个限定，可以写成以下这种格式表示实现多个接口：``T extends Comparable & Serializable``。

## 使用通配符
使用通配符可以大大简化开发，主要有三种通配符类型。
### 子类型限定符
以下形式: ``? extends SomeClass``的格式为子类型限定符，限定泛型的类型为**SomeClass**的子类，但是不确定是哪个子类。比如有基类A, B和C都继承A, 这个时候ArrayList&lt;? extends A&gt;类型的变量可以是ArrayList&lt;A&gt;, 可以是ArrayList&lt;B&gt;, 可以是ArrayList&lt;C&gt;。使用的限制是，这种类型的变量是只读的。因为不确定是哪个子类，所以不能擅自往里面写入。但是读取是可以的，因为已经明确了是A的子类类型，所以读出的值可以赋给A类型的变量。
### 超类型限定符
``? super SomeClass``的格式为超类型限定符，与上述相反，只能往此种类型写入SomeClass类型的变量，不能够从中读取，因为不知道是哪个具体的超类。
### ?通配符
无限定通配符读出的数据只能赋给Object类型的变量，写入是绝对不可以的，甚至null也是不允许的。《Java核心技术 卷一》给出了一个和null比较的示例用以说明用途，但是个人觉得还是很鸡肋。
## 使用的限制
泛型具有诸多的限制，Java泛型真的太鸡肋了，列举如下：

### 不能实例化泛型
容易理解，都不知道是什么类型，所以没办法调用构造函数，比如这样: ``T t = new T()``
### 不能直接创建泛型数组，以及泛型类的数组
不能这样创建数组：

    // 错误的示例
    T a = new T[10];
    List<T>[] b = new ArrayList<T>[];
    List<String>[] c = new ArrayList<String>[];
不能这样做的原因参见《Effective Java》的第25条建议。因为数组是支持**协变**的，而泛型是不支持的，所以如果允许这么做的话会破坏设计的初衷，得时刻提防着ClassCastException。如果想用泛型的数组，考虑用现有的泛型集合来替代。

泛型数组并不是绝对不能创建，以下代码是合法的：

    public static <T> T[] getArray(Class<T> type, int size) {
        return (T[]) new Object[size];
    } 
### 泛型类静态泛型数据无效
容易理解，静态数据只属于类。
### 注意擦除后的影响
擦除之后会讲到，以下代码编译期间不会通过：

    class A<T> {
        public boolean equals(T obj) {
            return super.equals(obj);
        }
    }
在进行泛型擦除之后，此方法变成``public boolean equals(Object obj) { ... }``此时类中就会有两个重名的方法，一个是擦除后的，一个是继承自Object类的，这时候就会引发冲突。
# 泛型虚拟机实现
## 擦除
Java的泛型比较鸡肋，泛型信息在编译之后就会被擦除了，例如下面的类：

    class A<T> {
        T field;

        public T getField() {
            return field;
        }

        public void setField(T field) {
            this.field = field;
        }
    }
经过擦除之后，会变成：

    class A {
        Object field;

        public Object getField() {
            return field;
        }

        public void setField(Object field) {
            this.field = field;
        }
    }
擦除之后变成原始的A类型，所以对于A<String>和A<Integer>在运行期间对应的都是同一个类。

对于泛型方法的调用，采取了强制类型转换，比如：

    A<String> a = new A<>();
    a.setField("hhh");
    String field = a.getField();
当擦除之后变成：

    A a = new A();
    a.setField((Object) "hhh");
    String field = (Object) a.getField();

泛型T经过擦除变成Object类型，如果加了泛型限定符的话，例如``T extends Comparable``，那么就会用Comparable类型来进行擦除。
## 桥方法
考虑下面一种情况：

    class A implements Comparable<A> {
        @Override
        public int compareTo(A o) {
            return 0;
        }
    }
经过擦除之后，接口中的方法变成了``public int compareTo(Object o)``，这时候A类中就会有两个方法，一个接受A类参数，一个接受Object类型参数。后者是编译器为此类生成的桥方法：

    public int compareTo(Object o) {
        return compareTo((A) o);
    }
对于返回类型来说，也会生成桥方法，但是此时可能会出现函数签名一致的情况，因为Java是通过函数名和参数列表来标记方法的，这时候会交给虚拟机进行正确的处理。

桥方法不仅用于泛型，还有很多方面用到了桥方法，例如子类重写父类方法的时候，返回类型可以更加的“严格”，这种机制也是通过桥方法来实现的。
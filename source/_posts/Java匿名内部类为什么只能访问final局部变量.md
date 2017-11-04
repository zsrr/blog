---
title: Java匿名内部类为什么只能访问final局部变量
date: 2017-11-04 19:08:55
tags: [Java]
categories: Java

---
黑Java的一篇……hhh

<!--more-->

# 匿名内部类的使用
使用方式一般是这样子的:

    public class Main {
        
        int c = 3;
        
        public void run() {
            final int a = 10;
            Interface i = new Interface() {
                @Override
                public void method() {
                    System.out.println(a);
                    System.out.println(c);
                    c = 10;
                }
            };
            i.method();
        }

        public static void main(String[] args) throws Exception {
            new Main().run();
        }
    }
匿名内部类可以访问类实例的成员，包括受保护的和私有的，此时不要求成员变量是final的。但是对于局部变量来说，它只能是final的，尽管在Java8中不用显式的声明，但还是要求局部变量是final的(真是神奇的设计啊hhh)。
# 原理
对于上面的代码，翻译给JVM的代码实际上是这样的：

    public class Main {

        int c = 3;
        
        private class SomeClass implements Interface {
            final int a;

            public SomeClass(int a) {
                this.a = a;
            }
            
            @Override
            public void method() {
                System.out.println(a);
                System.out.println(c);
                c = 10;
            }
        }

        public void run() {
            int a = 10;
            SomeClass s = new SomeClass(a);
            s.method();
        }

        public static void main(String[] args) throws Exception {
            new Main().run();
        }
    }
其中SomeClass是我随便命名的，实际情况不是这样的。经过编译之后，匿名内部类会翻译成一个和方法同级别的内部类，对于访问的局部变量，通过构造函数传给内部类实例。所以在第一份代码访问的a实际上不是之前定义的局部变量a。这就是为什么要规定final的原因了：**如果不规定是final的话，那么内部类对变量的写入对外部是不可见的，这就造成了内外数据不一致的问题。**

额，我真的佩服Java对匿名内部类的处理……

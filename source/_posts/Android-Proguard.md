---
title: Android ProGuard
date: 2017-04-24 11:06:25
tags: [Android,Java]
categories: 安卓开发

---
因为项目里面一个开源库debug的包可以而release的包运行不起来，查了半天原来是混淆的锅，赶紧补
<!--more-->

先附上一个github地址： [android-proguard-snippets](https://github.com/krschultz/android-proguard-snippets)，里面记录了各种库的ProGuard规则。

# ProGuard简介
ProGuard是用来压缩，最优化，混淆，预检验Java代码的工具，能够防止class文件被反编译以及减小最终生成的程序的大小，其工作流程图如下图所示(图片来自官方文档):

![](https://www.guardsquare.com/files/media/guardsquare2016/Website/ProGuard/ProGuard_build_process.png)

ProGuard要求制定Library jars，以此来重新组织class文件的依赖关系，Library jars通常是保持不变的。

代码的某些地方不能被混淆，如main方法，这些地方称为入口点(Entry Points)。反射时用到的类、方法、属性名也不能被混淆。

# ProGuard的使用
先附上下载地址：[ProGuard 5.3.3下载地址](http://ok34fi9ya.bkt.clouddn.com/proguard5.3.3.zip)

下载完成之后切换到bin目录，在终端执行：

	sh proguard.sh [options]
也可以运行proguardgui.sh来开启图形界面。
## ProGuard Options
options通常写在一个单独的configuration file里面，并且可以递归地读取配置文件，下面来介绍几个最常见的Options。
### Input/Output Options
#### -inlcude *filename*
读取配置文件，可以在配置文件中声明表示递归地读取。
#### -basedirectory *directoryname*
声明配置文件的相对位置。
#### -injars *class_path*
指定需要处理的jar文件。jar中的class文件将被处理，并将处理之后的class文件写到output jars。默认情况下非class文件将只进行复制处理。
#### -outjars *class_path*
指定output jars的名字，应该避免output jars重写掉input files。
#### -libraryjars *class_path*
指定应用的库文件，这些文件不会再output jars里面出现。被应用中的类继承的class需要被指定，用到的class不用被指定，但是指定它们有助于最优化代码。
#### -skipnonpubliclibraryclasses
用来指定处理Library jars的时候跳过非公有类，以此来节省ProGuard运行时所占的时间和空间，但是不能总是指定这个选项，例如当某些类库中的非公有类被公有类继承的时候。
#### -dontskipnonpubliclibraryclasses
上一个选项的反义词，从版本4.5开始这是默认选项。
#### -dontskipnonpubliclibraryclassmembers
指定不要忽略包级私有类的成员（属性和方法）。由于这些类不经常被应用程序所引用，因此忽略它们是ProGuard的默认选项，但是在某些情况下不能忽略，例如应用程序的类与它们处在同一个包中，这时候被引用的包级私有类不能被忽略。
#### -keepdirectories [*directory_filter*]
指定需要在Output jars里面保留的文件夹名。如果directory_filter为空，那么所有的文件夹都将被保存。
#### -target *version*
指定class文件的级别。
#### -forceprocessing
对class文件进行强制处理，即便输出看起来是过时的。

### Keep Options
#### -keep [,*modifier*,...] *class_specification*
指定类和类的成员当作入口点，例如一个Java应用程序的入口类以及main方法，对于类库来说，所有的public元素都应该当作入口点。此指令会保护类和其成员不被混淆和删除。
#### -keepclassmembers [,*modifier*,...] *class_specification*
指定保护类的成员，例如，实现了Serializable接口的用于序列化的类成员应该被保护。此指令会保证类成员不会被混淆和删除。
#### -keepclasseswithmembers [,*modifier*,...] *class_specification*
指定保护拥有指定类成员的类及指定类成员。如在一个Java程序中，要保护拥有main方法的类，可以用这个指令，而不用一一列举。
#### -keepnames *class_specification*
指定类的名称不会被混淆，前提是在压缩代码的时候此类仍然存在，仅在混淆阶段起作用。
#### -keepclassmembernames *class_specification*
指定类成员的名字不会被混淆。仅在混淆阶段起作用，前提是Shrink过后类成员仍然存在。
#### -keepclasseswithmembernames *class_specification*
指定拥有指定类成员的类及指定类成员的名字不会被混淆，如要保护一个拥有本地方法的类名和其本地方法的名字不被混淆。仅在混淆阶段起作用。

关于以上几个 -keep选项，详见下表：
![来自官方文档](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-26%20%E4%B8%8B%E5%8D%884.21.32.png)

### Shirinking Options（非重点）
####  -dontshrink
指定不压缩代码，默认情况下，除了被上述多样的 -keep选项保护起来的类和类成员，以及它们直接或者间接依赖的类不被压缩外，其余的都不会被保留。
### Optimization options（非重点）
#### -dontoptimize
指定不进行最优化，默认开启。
#### -optimizationpasses n
指定ProGuard进行对代码进行最优化的次数。
#### -optimizations
指定优化时所采取的算法，属于高级选项，关于optimizations filter，请查看：[optimizations](https://www.guardsquare.com/en/proguard/manual/optimizations)
#### -assumenosideeffects *class_specification*
指定无关紧要的方法，ProGuard会在优化阶段去除对此方法所有的调用，例如可以去除打log的方法调用，注意此举是危险的，除非我们能够确定一个方法在整个应用程序中是绝对没有用处的，否则不要轻易指定。
#### -mergeinterfacesaggressively
合并接口，能够减少最终输出类的数目，缩小体积。不过在某些Java虚拟机上会带来性能的损失。

**注：**Android Studio默认的配置文件最优化是不开启的，因为Android Java虚拟机对于优化和预检验的代码不友好。
### Obfuscation options
#### -dontobfuscate
指定关闭混淆，混淆默认是开启的，除了被 -keep选项保护起来的类和类成员，其余的类和类成员都被赋予了随机的名字。
#### -applymapping *filename*
重新利用指定file中的映射集，在此文件中列出的类和类成员被赋予映射集文件中指定的名字，没有在其中列举的但出现在输入中的类和类成员被赋予新的名字。
#### -overloadaggressively
允许重载，很多类属性和方法会得到相同的名称，只要方法的参数名不同或者返回值不同（遵守Java的约定），这会使得输出的体积更小，不过很多虚拟机并不支持此选项，小心使用。
#### -useuniqueclassmembernames
为名字相同的类成员指派相同的混淆名
#### -dontusemixedcaseclassnames
指定不用大小写混合的混淆名，这在大小写不敏感的操作系统（如Mac OS）上尤其有用。
#### -keeppackagenames [*package_filter*]
指定的包名不会被混淆。
#### -keepattributes [*attribute_filter*]
指定不会被混淆的属性，具体支持的属性，请查看：[属性列表](https://www.guardsquare.com/en/proguard/manual/attributes) 

例如，为了便于调试，需要保存异常信息，则需要以下选项：

	-keepattributes SourceFile,LineNumberTable
#### -keepparameternames
保证方法的参数名不被混淆。

### Preverification options
Android Java虚拟机对其支持的不友好，默认关闭，在此不再列举该选项。

**以上内容大部分为常用的，ProGuard的用法实在是太多了，没有必要在这里一一列举，混淆和保存选项是重要内容，其余请查看官方文档。**
### class_specification
用来指定类和其成员的语句，用在 -keep选项和 -assumenosideeffects选项当中，模板设计的风格非常像Java，还有很多通配符，具体的格式如下：

	[@annotationtype] [[!]public|final|abstract|@ ...] [!]interface|class|enum classname
    [extends|implements [@annotationtype] classname]
	[{
	    [@annotationtype] [[!]public|private|protected|static|volatile|transient ...] <fields> |
	                                                                      (fieldtype fieldname);
	    [@annotationtype] [[!]public|private|protected|static|synchronized|native|abstract|strictfp ...] <methods> |
	                                                                                           <init>(argumenttype,...) |
	                                                                                           classname(argumenttype,...) |
	                                                                                           (returntype methodname(argumenttype,...));
	    [@annotationtype] [[!]public|private|protected|static ... ] *;
	    ...
	}]
方括号[]代表内部的内容是可选的。

...代表前面几个是可以进行多个选择的。

classame必须写全称，例如java.lang.String。对于内部类来讲，类名前面和外部类后面要有"$"分隔。例如：`java.lang.Thread$State`。classname里面可以有如下的通配符：

- ?：匹配任意一个字符。
- *：匹配任何不带.的语句，不能够匹配带有.号的包名。
- **：可以匹配带有.的语句，专门用来匹配带有.的包名。

@annotationtype代表限制被注解类型注释的类，annotationtype和上述classname的规则是一样的。

属性和方法名大部分和Java中是一样的，除了方法中参数列表只包含参数类型不包含参数名称，属性和方法名还可以包含以下通配符：

- &lt;init&gt; 匹配所有的构造函数。
- &lt;fields&gt; 匹配所有的属性。
- &lt;methods&gt; 匹配所有的方法。
- * 匹配所有的方法和属性。

类型描述符可以包含以下通配符：

- % 匹配任何值类型，如boolean, int等等。
- ？ 匹配引用类型中的单个字符。
- * 匹配不带.的引用类型。
- ** 匹配带有包名的引用类型，但不匹配数组类型。
- *** 匹配任何类型，包括值类型和引用类型，以及数组类型。
- ... 匹配任意数量的任意类型。
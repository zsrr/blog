---
title: Mac OS Sierra下编译Android 7.1.1
date: 2017-02-16 19:46:37
tags: Android
categories: 安卓开发

---
# 为什么要编译源码
安卓进阶的最好方式就是Read the fucking source code!不去研究源码的程序员不是好程序员，呵呵呵……好吧，其实笔者要编译源码的原因是因为想研究DataBinding源码结果发现自带的sdk看不到核心逻辑，而笔者却是一个处女座……
# 流程
## 配置repo
repo是google为了方便Android源代码的管理，用一系列Python脚本封装git命令做成的工具，具体的请看：[Repo的使用](http://ticktick.blog.51cto.com/823160/1653304)  

安装代码两行：

	$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
	$ chmod a+x ~/bin/repo
## 配置JDK
Android 7.1.1需要Java8，到官方下载安装即可：[传送门](http://www.oracle.com/technetwork/java/javase/downloads/index-jsp-138363.html)

Java环境变量配置：

	export JAVA_HOME=`/usr/libexec/java_home -v 1.8`
	export PATH=$JAVA_HOME/bin:$PATH 
	export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
## 安装curl
Mac OS自带的curl不带有openssl，不满足Jack编译的条件，因此需要用brew手动下载并替换默认的curl，命令如下：

	$ brew install curl --with-openssl
	$ export PATH=$(brew --prefix curl)/bin:$PATH
## 下载配置Mac OS sdk
由于Mac OS 10.12sdk弃用了syscall函数，所以我们需要旧版本的sdk：[下载地址](https://github.com/phracker/MacOSX-SDKs/releases)

下载好之后解压（如果下载下来的文件格式是xz文件那就不能用系统默认的解压工具），将解压得到的文件夹放在**/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs**下，大功告成。
## 创建大小写敏感的磁盘区域
由于Mac OS默认不区别大小写，为了防止之后下载的文件冲突，需要新建一块大小写敏感的磁盘区域：

	$ hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 85g ~/android.dmg
大小尽量设置的大些，笔者的源码下载并且编译之后整个的大小是83g。创建完毕后即可在Finder中双击加载。
## 下载镜像文件
由于众所周知的原因，google官方的景象并不能有很好的体验，所以要用[清华大学的aosp镜像](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)，这里采用其官方推荐的做法，先下载一个每月更新的[初始化包](https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar)（为求速度和稳定性请用迅雷）。

下载完成后，将压缩包剪切到先前已经加载的磁盘区域当中，用终端cd到压缩包所在的目录，运行如下指令：

	$ tar xf aosp-latest.tar
	$ cd aosp
	$ repo sync
接下来便会等待一段时间，此时repo正将下载好的分支更新到最新，所以仍需下载。如果出现下载速度过慢，远程连接经常hang up，那就请为git设置代理，笔者采用的是ss代理，所以相应的设置如下：

	git config --global http.proxy 'socks5://127.0.0.1:1080'
	git config --global https.proxy 'socks5://127.0.0.1:1080'
## 开始编译
下载完后，用终端cd到aosp目录，做的第一件事就是加载封装好的命令集：

	$ source build/envsetup.sh
接着运行指令lunch，会给出很多选项，对应不同的设备，这里笔者只是想查看一下源码，并没有刷机的需求，所以我就选了aosp_arm-eng。

接下来运行：
	
	$ make -j4
注意j4是因为笔者的笔记本是二核心的，如果是4核心就是8，以此类推。

下面就是漫长的等待，大概3，4个小时左右吧。
## 导入Android Studio
编译完成之后，运行：

	$ mmm development/tools/idegen/
	$ development/tools/idegen/idegen.sh
如果出现找不到指令的错误，请重新加载指令集。

为保证AS是大小写敏感的，在导入AS之前，先编辑**/Applications/Android\ Studio.app/Contents/bin**下的**idea.properties**文件，在其最后加上如下代码：

	idea.case.sensitive.fs=true

这时候aosp目录之下会出现android.ipr文件，此文件便是AS所需。用AS打开该文件，又是一个漫长的等待过程，大概20分钟左右吧。

导入之后，为了防止我们在观察源码时跳转到不是我们需要的.class文件，还需要做额外的几件事情：
### 配置JDK和SDK
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-16%20%E4%B8%8B%E5%8D%889.38.12.png)
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-16%20%E4%B8%8B%E5%8D%889.40.01.png)
### 删除jar依赖
在项目设置(command + ;)中点击Modules，删除所有的jar包依赖，再添加两个文件夹依赖，分别是aosp文件夹下的external和frameworks文件夹。
### 注释掉android.iml文件的&lt;sourceFloder&gt;标签
上面两步之后，一般情况下可以愉快的看源码了，可是有时候还会出现跳转错误，这是因为导入的其他的文件夹里面的类与external和frameworks冲突所致，这时候就需要编辑aosp目录下的android.iml文件，将发生冲突的文件夹的位置记录下来，并在iml文件中将&lt;sourceFolder&gt;标签注释掉。
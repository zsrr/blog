---
title: Maven远程部署Tomcat8
date: 2017-08-30 12:03:51
tags: [Maven,Tomcat]
categories: Tomcat

---
昨天第一次远程部署踩了不少坑，今天记录一下部署的具体过程及注意事项，Tomcat的版本是8.5.20，Maven版本是3.5.0。

<!--more-->

# 远程服务器配置
在部署之前需要先对Tomcat服务器进行配置。

## 配置管理员用户
需要先在Tomcat安装目录下/conf/tomcat-users.xml配置管理员用户，赋予相应的权限：

	<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
              
	  <role rolename="manager-gui"/>
	  <role rolename="manager-script"/>
	  <user username="admin" password="你自己的密码" roles="manager-gui,manager-script"/>
	</tomcat-users>
## 开启manager的远程访问
从Tomcat8.5之后开始，默认只有本机可以访问相应的manager的URL，因此需要开启远程用户的访问：

	cd webapps/manager/META-INF/
	vi context.xml
将以下一行注释掉：

	<Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
         
上两步配置完成之后要重新启动Tomcat。

# 本地Maven配置
在项目的pom.xml中加入：

	<plugins>
        <plugin>
            <groupId>org.apache.tomcat.maven</groupId>
            <artifactId>tomcat7-maven-plugin</artifactId>
            <version>2.2</version>
            <configuration>
                <server>tomcat7</server>
                <path>/bangbang</path>
                <url>http://你自己的远程服务器域名:8080/manager/text</url>
            </configuration>
        </plugin>
    </plugins>
然后在用户的Maven目录下更改settings.xml，在&lt;servers&gt;节点下加入:

    <server>
      <id>tomcat7</id>
      <username>之前配置的远程管理员用户名</username>
      <password>之前配置的远程管理员密码</password>
    </server>
此时在项目文件下执行:

	mvn tomcat7:redeploy
即可发布。

成功的标志：

	[INFO] tomcatManager status code:200, ReasonPhrase:
	[INFO] OK - Deployed application at context path [/***]
	[INFO] ------------------------------------------------------------------------
	[INFO] BUILD SUCCESS
	[INFO] ------------------------------------------------------------------------
	[INFO] Total time: 14.339 s
	[INFO] Finished at: 2017-08-30T12:17:43+08:00
	[INFO] Final Memory: 19M/204M
	[INFO] ------------------------------------------------------------------------
# 需要注意的问题
笔者在发布的过程中遇到如下问题，即发布之后远程访问404，此时maven执行后的信息为：

	[INFO] tomcatManager status code:200, ReasonPhrase:
	[INFO] FAILED - Deployed application at context path [/***] but context failed to start
	[INFO] ------------------------------------------------------------------------
	[INFO] BUILD SUCCESS
	[INFO] ------------------------------------------------------------------------
	[INFO] Total time: 14.339 s
	[INFO] Finished at: 2017-08-30T12:17:43+08:00
	[INFO] Final Memory: 19M/204M
	[INFO] ------------------------------------------------------------------------
虽然下面显示BUILD SUCCESS，但要注意上面的信息：

	[INFO] FAILED - Deployed application at context path [/***] but context failed to start
此时可以切换到Tomcat的目录查看Log：

	cd logs
	cat localhost.具体日期.log
发现原来是没有为Tomcat服务器加入MySQL connector jar导致的……当然这是笔者自己的报错信息，具体还要根据日志排查问题。

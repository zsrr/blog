---
title: hexo + 阿里云ECS(CentOS7)搭建个人博客
date: 2017-07-06 16:50:56
tags: [配置]
categories: 环境搭建

---
为纪念今日网站通过审核正式成立，特此作博文一篇。
<!--more-->

# 环境
笔者的环境为hexo + 腾讯云域名 + 阿里云学生认证购买的CentOS7 ECS主机，本地环境为Mac OS。
# 为云服务器搭建nginx
在本地用ssh登录远程服务器，服务器的地址为其公网ip，可以在控制台的云服务器实例查看。

登录到远程服务器之后，先配置nginx源

	cd /etc/yum.repos.d
	vi nginx.repo
	
	[nginx]
	name=nginx repo
	baseurl=http://nginx.org/packages/centos/7/$basearch/
	gpgcheck=0
	enabled=1

配置完毕之后，执行：

	yum install nginx -y
安装完毕之后，执行：

	// 开启nginx
	systemctl start nginx
	// 开机重启自动开启nginx
	systemctl enable nginx
用以下命令查看80端口的占用情况：

	netstat -anp|grep 80
若结果中有

	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      689/nginx: master p
则说明nginx启动成功。
# 开启远程服务器的80端口
笔者在这里困惑了好久，我认为80端口是默认打开的，直接输入服务器公网ip便可访问nginx默认页面，但是阿里云的防火墙默认是不开启80端口的，需要进行配置。

打开控制台 -> 云服务器实例 -> 更多 -> 安全组配置
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-06%20%E4%B8%8B%E5%8D%885.26.06.png)

点击配置规则 -> 添加安全组规则，http访问进行如下设置：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-06%20%E4%B8%8B%E5%8D%885.30.05.png)
保存并退出，这时访问服务器公网ip就能看到默认的nginx页面了。
# 配置域名解析
需要将域名配置到远程服务器上，打开域名的控制中心，添加如下配置：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-06%20%E4%B8%8B%E5%8D%885.33.53.png)
其中记录值填写你自己的服务器公网ip地址，大约十分钟后解析才会生效。
# 本地发布
接下来就是发布了。先远程登录服务器，配置发布的位置和nginx解析，在这里笔者直接在根目录下创建了/stephenzhang/blog作为静态页面的存储地址。

配置nginx:

	vi /etc/nginx/conf.d/default.conf
	
	server {
	    listen       80;
	    server_name  www.stephenzhang.me; # 你自己的域名
	    root /stephenzhang/blog; # 根目录地址
	    index index.html index.htm;
	
	    location / {
	        try_files $uri $uri/ /index.html;
	    }
	    
	    ...
    }
保存并重启nginx。

本地安装hexo，至于hexo的基本使用我就不想再说了，直接去官网看文档。在相应的目录下安装rsync：

	npm install hexo-deployer-rsync --save
修改_config.yml的deploy节点，如下所示：

	deploy:
	  type: rsync
	  host: 47.94.245.43 # 远程主机公网ip
	  user: root # 远程主机用户名
	  root: /stephenzhang/blog # 远程主机发布根目录
这时执行hexo d输入远程服务器登录密码之后便可执行发布，注意最后的根目录要在远程服务器中存在才可。

# 审核
故事还没有结束，得先到工信局备案才行，备案过程大约1个星期左右，过后便可愉快使用了。
	

title: 一步一步用jenkins，ansible，supervisor打造一个web构建发布系统
date: 2015-02-12 14:55:04
tags:
 - jenkins
 - ansible
 - supervisor
 - docker
 - devops
 - 运维

categories:
 - 技术
 
---

一步一步用jenkins，ansible，supervisor打造一个web构建发布系统。

本来应该还有gitlab这一环节的，但是感觉加上，内容会增加很多。所以直接用github上的spring-mvc-showcase项目来做演示。

https://github.com/spring-projects/spring-mvc-showcase

本文的环境用docker来构建。当然也可以任意linux环境下搭建。

如果没有安装docker，可以参考官方的文档：
https://docs.docker.com/installation/ubuntulinux/#ubuntu-trusty-1404-lts-64-bit 

下面将要介绍的完整流程是：

- github作为源代码仓库
- jenkins做为打包服务器，Web控制服务器
- ansible把war包，发布到远程机器
   1. 安装python-pip
   2. 用pip安装supervisor
   3. 安装jdk
   4. 下载，部署tomcat
   5. 把tomcat交由supervisor托管
   6. 把jenkins生成的war包发布到远程服务器上
   7. supervisor启动tomcat
   8. 在http端口等待tomcat启动成功
- supervisor托管app进程，提供一个web界面可以查看进程状态，日志，控制重启等。

在文章的最后，会给出一个完整的docker镜像，大家可以自己运行查看实际效果。

## 安装jenkins
- 先用docker来启动一个名为“jenkins”的容器：

```bash
sudo docker run -i -t -p 8080:8080 -p 8101:8101 -p 9001:9001 --name='jenkins' ubuntu /bin/bash
```
8080是jenkins的端口，8101是spring-mvc-showcase的端口，9001是supervisor的web界面端口

执行完之后，会得到一个container的shell。接着在这个shell里安装其它组件。

- 安装open jdk 和 git：

```bash
sudo apt-get update
sudo apt-get install openjdk-7-jdk git
```
- 下载配置tomcat：

```
apt-get install wget
mkdir /opt/jenkins
cd /opt/jenkins
wget http://apache.fayea.com/tomcat/tomcat-8/v8.0.18/bin/apache-tomcat-8.0.18.tar.gz
tar xzf apache-tomcat-8.0.18.tar.gz

```
- 安装jenkins：

```bash
cd /opt/jenkins/apache-tomcat-8.0.18/webapps
wget http://mirrors.jenkins-ci.org/war/latest/jenkins.war
rm -rf ROOT*
mv jenkins.war ROOT.war
```
- 启动jenkins：

```
/opt/jenkins/apache-tomcat-8.0.18/bin/startup.sh
```
然后在本机用浏览器访问：http://localhost:8080/ ，可以看到jenkins的界面了。

## 配置jenkins
### 安装git插件
安装git插件：
https://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin

在“系统管理”，“插件管理”，“可选插件”列表里，搜索“Git Plugin”，这样比较快可以找到。

因为jenkins用google来检查网络的连通性，所以可能在开始安装插件时会卡住一段时间。

### 配置maven, java
打开 http://localhost:8080/configure，
在jenkins的系统配置里，可以找到maven，git，java相关的配置，只要勾选了，在开时执行job时，会自动下载。
![jenkins-config-maven](/img/jenkins-config-maven.png)

JDK可以选择刚才安装好的openjdk，也可以选择自动安装oracle jdk。
![jenkins-config-openjdk7](/img/jenkins-config-openjdk7.png)

Git会自动配置好。


## 配置ssh服务
安装sshd服务：
```bash
sudo apt-get install openssh-server sshpass
```
编辑
vi /etc/ssh/sshd_config
把
```
PermitRootLogin without-password
```
改为：
```
PermitRootLogin yes
```
重启ssh服务：
```bash
sudo /etc/init.d/ssh restart
```
为root用户配置密码，设置为12345：
```bash
passwd
```
最后尝试登陆下：
```bash
ssh root@127.0.0.1
```

## 安装ansible
在jenkins这个container里，继续安装ansible，用来做远程发布用。

先安装pip，再用pip安装ansible：
```bash
sudo apt-get install python-pip python-dev build-essential git
sudo pip install ansible
```

## 配置ansible playbook
把自动发布的ansible playbook clone到本地：

https://github.com/hengyunabc/jenkins-ansible-supervisor-deploy
```bash
mkdir -p /opt/ansible
cd /opt/ansible
git clone https://github.com/hengyunabc/jenkins-ansible-supervisor-deploy
```

## 在jenkins上建立deploy job
- 新建一个maven的项目/job，名为spring-mvc-showcase
![jenkins-new-job-maven](/img/jenkins-new-job-maven.png)


- 在配置页面里，勾选“参数化构建过程”，再依次增加“String”类型的参数

![jenkins-new-job-parameters](/img/jenkins-new-job-parameters.png)

共有这些参数：
```
	app	   要发布的app的名字
 	http_port	  tomcat的http端口
 	https_port	tomcat的https端口
 	server_port	tomcat的server port
 	JAVA_OPTS	 tomcat启动的Java参数
 	deploy_path	  tomcat的目录
 	target_host	 要发布到哪台机器
 	war_path	   jenkins生成的war包的目录
```

- “源码管理”，选择Git，再填入代码地址

https://github.com/spring-projects/spring-mvc-showcase.git

![jenkins-new-job-git](/img/jenkins-new-job-git.png)

- 在“Post Steps”里，增加调用ansible playbook的shell命令
![jenkins-new-job-shell-ansible](/img/jenkins-new-job-shell-ansible.png)

```bash
cd /opt/ansible/jenkins-ansible-supervisor-deploy
ansible-playbook -i hosts site.yml --verbose --extra-vars "target_host=$target_host app=$app http_port=$http_port https_port=$https_port server_port=$server_port deploy_path=$deploy_path JAVA_HOME=/usr JAVA_OPTS=$JAVA_OPTS deploy_war_path=$WORKSPACE/$war_path"
```
最后，保存。

## 测试构建
一切都配置好之后，可以在jenkins界面上，在左边，选择“Build with Parameters”，“开始”来构建项目了。

如果构建成功的话，就可以打开 http://localhost:8101 ，就可以看到spring-mvc-showcase的界面了。

![spring-mvc-showcase-webui](/img/spring-mvc-showcase-webui.png)

打开 http://localhost:9001 可以看到superviosr的控制网页，可以查看tomcat进程的状态，重启，查看日志等。

![supervisor-webui](/img/supervisor-webui.png)

如果想要发布到其它机器上的话，只要在

  ``` 
  /opt/ansible/jenkins-ansible-supervisor-deploy/hosts 
  ``` 
  
文件里增加相应的host配置就可以了。

## 其它的一些东东
如果提示
```
to use the 'ssh' connection type with passwords, you must install the sshpass program
```
则安装：
```bash
sudo apt-get install sshpass
```

## 演示的docker image
如果只是想查看实际运行效果，可以直接把 hengyunabc/jenkins-ansible-supervisor 这个image拉下来，运行即可。

```bash
docker run -it -p 8080:8080 -p 8101:8101 -p 9001:9001 --name='jenkins' hengyunabc/jenkins-ansible-supervisor
```

## 总结

- jenkins提供了丰富的插件，可以定制自己的打包这过程，并可以提供完善的权限控制
- ansible可以轻松实现远程部署，配置环境等工作，轻量简洁，功能强大
- supervisor托管了tomcat进程，提供了web控制界面，所有运行的程序一目了然，很好用
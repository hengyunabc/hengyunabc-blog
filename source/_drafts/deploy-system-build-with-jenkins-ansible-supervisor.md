title: deploy-system-build-with-jenkins-ansible-supervisor
date: 2015-02-12 14:55:04
tags:
 - jenkins
 - ansible
 - supervisor
 - devops
---

一步一步用jenkins，ansible，supervisor打造一个web构建发布系统。

本文的环境用docker来构建。当然也可以任意linux环境下搭建。

如果没有安装docker，可以参考官方的文档：
https://docs.docker.com/installation/ubuntulinux/#ubuntu-trusty-1404-lts-64-bit 

##安装jenkins
- 先用docker来启动一个名为“jenkins”的容器：

```bash
sudo docker run -i -t -p 8080:8080 --name='jenkins' ubuntu /bin/bash
```
执行完之后，会得到一个container的shell。接着在这个shell里安装其它组件。

- 安装oracle jdk：

```bash
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```
- 安装tomcat：

```
apt-get install wget
apt-get install git
cd /opt
mkdir jenkins
cd jenkins
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

##安装ansible
在jenkins这个container里，继续安装ansible，用来做远程发布用。
先安装pip，再用pip安装ansible：
```bash
sudo apt-get install python-pip python-dev build-essential 
sudo pip install ansible
```

## 安装supervisor，这里貌似不对，改为用ansible安装！！
```bash
sudo pip install supervisor
```
supervisor然后用官方的脚本来启动：
https://github.com/Supervisor/initscripts

supervisor的模板需要 配置 http可以访问，还有密码
还有样例的 配置文件



##启动另一个container，发布的目标机器
如果没有空闲的资源，直接在同一个container或者同一台机器上执行也是可以的。
```bash
sudo docker run -i -t -p 8090:8090 --name='deployHost' ubuntu /bin/bash
```
安装sshd服务：
```bash
sudo apt-get install openssh-server
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
再查看到这个container的ip：
```
ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/'
```
比如，我启动的这个containe的ip是```172.17.0.18```。

##配置ansible playbook
安装supervisor
下载tomcat，解压，
配置一个发布spring-mvc-showcase的例子

改为发布到本机？？ 这样一个container就可以搞定了，不用搞多个。。

##配置jenkins job



##其它的一些东东
如果提示
```
to use the 'ssh' connection type with passwords, you must install the sshpass program
```
则安装：
```bash
sudo apt-get install sshpass
```
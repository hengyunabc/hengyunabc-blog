title: kafka manager安装
date: 2015-03-02 15:19:26
tags:
 - kafka
 - java

  
categories:
 - 技术
 
---

##项目信息

https://github.com/yahoo/kafka-manager

这个项目比  https://github.com/claudemamo/kafka-web-console 要好用一些，显示的信息更加丰富，kafka-manager本身可以是一个集群。

不过kafka-manager也没有权限管理功能。

Kafka web console的安装可以参考之前的blog：

http://blog.csdn.net/hengyunabc/article/details/40431627

##安装sbt
sbt是scala的打包构建工具。

http://www.scala-sbt.org/download.html

ubuntu下安装：
```bash
echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
sudo apt-get update
sudo apt-get install sbt
```

##下载，编绎

编绎，生成发布包：

```bash
git clone https://github.com/yahoo/kafka-manager
cd kafka-manager
sbt clean dist
```

生成的包会在kafka-manager/target/universal 下面。生成的包只需要java环境就可以运行了，在部署的机器上不需要安装sbt。

如果打包很慢的话，可以考虑配置代理。

##部署
打好包好，在部署机器上解压，修改好配置文件，就可以运行了。
- 解压

```bash
unzip kafka-manager-1.0-SNAPSHOT.zip
```
- 修改conf/application.conf，把kafka-manager.zkhosts改为自己的zookeeper服务器地址

```
kafka-manager.zkhosts="localhost:2181"
```
- 启动

```bash
cd kafka-manager-1.0-SNAPSHOT/bin
./kafka-manager -Dconfig.file=../conf/application.conf
```
查看帮助 和 后台运行：
```bash
./kafka-manager -h
nohup ./kafka-manager -Dconfig.file=../conf/application.conf >/dev/null 2>&1 &  
```

默认http端口是9000，可以修改配置文件里的http.port的值，或者通过命令行参数传递：
```bash
./kafka-manager -Dhttp.port=9001 
```

正常来说，play框架应该会自动加载conf/application.conf配置里的内容，但是貌似这个不起作用，要显式指定才行。

参考： https://github.com/yahoo/kafka-manager/issues/16



##sbt 配置代理
sbt的配置http代理的参考文档：

http://www.scala-sbt.org/0.12.1/docs/Detailed-Topics/Setup-Notes.html#http-proxy

通过-D设置叁数即可：
```bash
java -Dhttp.proxyHost=myproxy -Dhttp.proxyPort=8080 -Dhttp.proxyUser=username -Dhttp.proxyPassword=mypassword
```

也可以用下面这种方式，设置一下SBT_OPTS的环境变量即可：
```bash
export SBT_OPTS="$SBT_OPTS -Dhttp.proxyHost=myproxy -Dhttp.proxyPort=myport"
```
要注意的是，myproxy，这个值里不要带http前缀，也不要带端口号。

比如，你的代理是http://localhost:8123，那么应该这样配置：
```bash
export SBT_OPTS="$SBT_OPTS -Dhttp.proxyHost=localhost -Dhttp.proxyPort=8123"
```

##打好的一个包

如果打包有问题的小伙伴可以从这里下载：

http://pan.baidu.com/s/1kTtFpGV

md5： bde4f57c4a1ac09a0dc7f3f892ea9026
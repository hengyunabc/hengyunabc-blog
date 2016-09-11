title: 利用github搭建个人maven仓库
date: 2015-08-05 21:32:05
tags:
 - github
 - maven
 - java

categories:
 - 技术


---

## 缘起

之前看到有开源项目用了github来做maven仓库，寻思自己也做一个。研究了下，记录下。

简单来说，共有三步：

1. deploy到本地目录
1. 把本地目录提交到gtihub上
1. 配置github地址为仓库地址


## 配置local file maven仓库

### deploy到本地

maven可以通过http, ftp, ssh等deploy到远程服务器，也可以deploy到本地文件系统里。例如：

```xml
  <distributionManagement>
    <repository>
      <id>hengyunabc-mvn-repo</id>
      <url>file:/home/hengyunabc/code/maven-repo/repository/</url>
    </repository>
  </distributionManagement>
```
通过命令行则是：
```bash
mvn deploy -DaltDeploymentRepository=hengyunabc-mvn-repo::default::file:/home/hengyunabc/code/maven-repo/repository/
```
**推荐使用命令行来deploy，避免在项目里显式配置。**

https://maven.apache.org/plugins/maven-deploy-plugin/

https://maven.apache.org/plugins/maven-deploy-plugin/deploy-file-mojo.html 

## 把本地仓库提交到github上
上面把项目发布到本地目录home/hengyunabc/code/maven-repo/repository里，下面把这个目录提交到github上。

在Github上新建一个项目，然后把home/hengyunabc/code/maven-repo下的文件都提交到gtihub上。

```bash
cd /home/hengyunabc/code/maven-repo/
git init
git add repository/*
git commit -m 'deploy xxx'
git remote add origin git@github.com:hengyunabc/maven-repo.git
git push origin master
```

最终效果可以参考我的个人仓库：

https://github.com/hengyunabc/maven-repo

## github maven仓库的使用
因为github使用了raw.githubusercontent.com这个域名用于raw文件下载。所以使用这个maven仓库，只要在pom.xml里增加：
```xml
    <repositories>
        <repository>
            <id>hengyunabc-maven-repo</id>
            <url>https://raw.githubusercontent.com/hengyunabc/maven-repo/master/repository</url>
        </repository>
    </repositories>
```


## maven仓库工作的机制
下面介绍一些maven仓库工作的原理。典型的一个maven依赖下会有这三个文件：

https://github.com/hengyunabc/maven-repo/tree/master/repository/io/github/hengyunabc/mybatis-ehcache-spring/0.0.1-SNAPSHOT
```
maven-metadata.xml
maven-metadata.xml.md5
maven-metadata.xml.sha1
```

maven-metadata.xml里面记录了最后deploy的版本和时间。

```xml

<?xml version="1.0" encoding="UTF-8"?>
<metadata modelVersion="1.1.0">
  <groupId>io.github.hengyunabc</groupId>
  <artifactId>mybatis-ehcache-spring</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <versioning>
    <snapshot>
      <timestamp>20150804.095005</timestamp>
      <buildNumber>1</buildNumber>
    </snapshot>
    <lastUpdated>20150804095005</lastUpdated>

  </versioning>
</metadata>

```

其中md5, sha1校验文件是用来保证这个meta文件的完整性。

maven在编绎项目时，会先尝试请求maven-metadata.xml，如果没有找到，则会直接尝试请求到jar文件，在下载jar文件时也会尝试下载jar的md5, sha1文件。

maven-metadata.xml文件很重要，如果没有这个文件来指明最新的jar版本，那么即使远程仓库里的jar更新了版本，本地maven编绎时用上-U参数，也不会拉取到最新的jar！

所以并不能简单地把jar包放到github上就完事了，一定要先在本地Deploy，生成maven-metadata.xml文件，并上传到github上。

参考：http://maven.apache.org/ref/3.2.2/maven-repository-metadata/repository-metadata.html

## maven的仓库关系

https://maven.apache.org/repository/index.html

![maven-repositories.png](/img/maven-repositories.png)

## 配置使用本地仓库
想要使用本地file仓库里，在项目的pom.xml里配置，如：

```xml
	<repositories>
		<repository>
			<id>hengyunabc-maven-repo</id>
			<url>file:/home/hengyunabc/code/maven-repo/repository/</url>
		</repository>
	</repositories>
```

## 注意事项
maven的repository并没有优先级的配置，也不能单独为某些依赖配置repository。所以如果项目配置了多个repository，在首次编绎时，会依次尝试下载依赖。如果没有找到，尝试下一个，整个流程会很长。

所以尽量多个依赖放同一个仓库，不要每个项目都有一个自己的仓库。


## 参考
http://stackoverflow.com/questions/14013644/hosting-a-maven-repository-on-github/14013645#14013645
http://cemerick.com/2010/08/24/hosting-maven-repos-on-github/
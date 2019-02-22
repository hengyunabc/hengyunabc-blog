---
title: Arthas Github Star破万啦，回顾开源历程，展望未来
date: 2019-02-20 20:58:28
tags:
 - arthas
 - java
 - github


categories:
  - 技术

---

## 一、Arthas的历史

![Arthas](https://alibaba.github.io/arthas/_images/arthas.png)

* https://github.com/alibaba/arthas

`Arthas`在阿里巴巴内部起源于2015年，当时微服务方兴未艾，我们团队一方面专注Spring Boot落地，提高开发效率，另外一方面，希望可以提高线上排查问题的能力和效率。当时我们经过选型讨论，选择基于`Greys`来开发，提供更好的应用诊断体验。

我们在用户体验上做了大量的改进：彩色UI，Web Console，内网一键诊断等。下面是内网一键在线诊断的截图，很多同事线上诊断问题的必备工具：

![image](https://user-images.githubusercontent.com/1683936/53077314-b0a5cd80-352c-11e9-99c7-25e5561c0ec0.png)


## 二、Arthas开源之后的工作

尽管`Arthas`在阿里内部广受好评，但它只是内部工具，很多离职同事都在一些文章里提到。做为Java开发的一员，我们使用到了很多开源的代码和工具，我们也希望可以提升广大开发人员的诊断效率，因此在2018年9月底，我们正式开源了Arthas。

在开源之后，`Arthas`多次登顶Github Trending，还被`@Java`官方twitter转发：

![image](https://user-images.githubusercontent.com/1683936/53077222-7fc59880-352c-11e9-837f-14948ac7c948.png)


在开源之后，`Arthas`发布了3个release版本，做了大量的改进：

* 全新的LOGO
* arthas-boot统一跨平台体验
* Arthas在线教程
* 全新版本的Web Console
* 全新的中英文档，感谢社区的大力支持
* JDK11全面支持，lamda类支持
* Docker支持
* 灵活的ognl命令
* 增加内存编译器，实现jad/mc/redefine一条龙
* Q键退出，history匹配，快捷键支持
* 不断完善的自动补全支持
* 重构重定向的支持


目前，`Arthas` Github star数10000+，月下载量7000+。在开源中国2018开源软件排行榜里，`Arthas`获得国产新秀榜第一名。

这是对我们过去工作的支持和肯定，也是我们持续改进`Arthas`的动力。

![Arthas Github](https://user-images.githubusercontent.com/1683936/53077846-edbe8f80-352d-11e9-8764-6c97a0fc91ff.png)

## 三、感谢贡献者们

`Arthas`在开源以来，不断收获到国内外贡献者的支持，目前已有40+贡献者，非常感谢他们的工作：

![Arthas贡献者](https://opencollective.com/arthas/contributors.svg?width=890&button=false)

特别感谢`@Hearen`贡献了大部分的英文翻译，`@wetsion`重构了新版本的Web Console。

参与贡献： https://github.com/alibaba/arthas/blob/master/CONTRIBUTING.md

## 四、Arthas实践系列文章

做为Arthas的用户，我们在实践中积累了很多经验，总结为一系列的文章，希望对大家线上排查问题有帮助：

* [Arthas实践--jad/mc/redefine线上热更新一条龙](https://mp.weixin.qq.com/s/fsHhkwfE-vrQRkQvUL1JGA)
* [Alibaba Arthas实践--获取到Spring Context，然后为所欲为](https://mp.weixin.qq.com/s/PlCwMhEFdtgHZTNBaNVIRQ)
* [Arthas实践--快速排查Spring Boot应用404/401问题](https://mp.weixin.qq.com/s/RQmYur3m2AsXFiuLwUCDnw)
* [当Dubbo遇上Arthas：排查问题的实践](https://mp.weixin.qq.com/s/-jhSV86_2E7WYhXbtVpAGQ)
* [使用Arthas抽丝剥茧排查线上应用日志打满问题](https://mp.weixin.qq.com/s/boGS0VK45mZ_zT25K44S5Q)
* [深入Spring Boot：利用Arthas排查NoSuchMethodError](https://mp.weixin.qq.com/s/eCIDxM9lXYX0cM7LGgfn1w)

## 五、Arthas 4.0规划

* 提供一个新的字节码框架，名为`bytekit`
* 插件化支持
* view分层，支持Web白屏化

详细链接： https://github.com/alibaba/arthas/issues/536 ，同时希望大家可以提出建议和参与。



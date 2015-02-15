title: 在hexo里配置remarkjs
date: 2015-01-19 23:17:49
tags: 
  - remarkjs
  - hexo
---
remarkjs是一个比较流行的用md格式写slides的库 ：[https://github.com/gnab/remark](https://github.com/gnab/remark "")

打算在hexo里增加对remark的支持。但是hexo会把所以的source目录下的md后缀的文件全部转换为html。
这样就很蛋疼了。

研究了下，发现hexo支持html, xml等在文件最前面加上layout: false就不会转换加上hexo模板的内容。

```
layout: false
--------
```

但是这个配置对于md后缀的文件不起效。

所以只能修改md后缀为其它后缀了。

- 在source目录下创建一个slides的目录；
- 新建test.html
```
layout: false
--------
<!DOCTYPE html>
<html>
  <head>
    <title>Title</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <style type="text/css">
      @import url(http://fonts.googleapis.com/css?family=Yanone+Kaffeesatz);
      @import url(http://fonts.googleapis.com/css?family=Droid+Serif:400,700,400italic);
      @import url(http://fonts.googleapis.com/css?family=Ubuntu+Mono:400,700,400italic);

      body { font-family: 'Droid Serif'; }
      h1, h2, h3 {
        font-family: 'Yanone Kaffeesatz';
        font-weight: normal;
      }
      .remark-code, .remark-inline-code { font-family: 'Ubuntu Mono'; }
    </style>
    <script src="http://gnab.github.io/remark/downloads/remark-latest.min.js" type="text/javascript"></script>

  </head>


  <body>
    <script type="text/javascript">
      var slideshow = remark.create({
    sourceUrl: '/slides/test.mymd'
      })
    </script>
  </body>
</html>
```
- 新建test.mymd文件，里面是slide的内容：
```markdown

class: center, middle

# Hello World

---
```
-  重新deploy，```hexo deploy```
- 在浏览器里访问，[http://localhost:4000/slides/test.html](http://localhost:4000/slides/test.html "")
本站的测试例子：[http://hengyunabc.github.io/slides/test.html](http://hengyunabc.github.io/slides/test.html "")


hexo其实最好能提供一个exclude的配置，允许某些目录不处理。在github上提了一个issue：[https://github.com/hexojs/hexo/issues/991](https://github.com/hexojs/hexo/issues/991 "")
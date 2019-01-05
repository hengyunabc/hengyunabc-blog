---
title: 扯谈下UTF-8
date: 2012-08-28 21:58:28
tags:
 - java
 - utf8
 - unicode
 - python


categories:
  - 技术

---

## 前言

本来想翻译这篇文章的（作者是utf-8编码，golang发明者之一）：

* UTF-8: Bits, Bytes, and Benefits: http://research.swtch.com/utf8

一则翻译起来很痛苦，二则觉得这篇文章有些地方可能说得不是太明白，所以结合其它的一些东东扯谈下utf-8。

## Unicode

先扯谈下Unicode。

Unicode就是为每一个字符（各种语言的各种字符）分配一个数字。所以它实际上是一个表，记录了字符和数字的对应关系。

比如汉字“你”，对应的数字是20320，16进制是4F60。

目前Unicode的范围从 U+0000 到 U+10FFFF 。有UTF-8，UTF-16，UTF-32三种编码方式。

其中UTF-8对应1到4个8-bit，UTF-16对应1到2个16-bit，UTF-32对应1个32-bit。

下面这个表，很清晰地总结了各种编码方式（from: http://www.unicode.org/faq/utf_bom.html ）：


|Name|UTF-8|UTF-16|UTF-16BE|UTF-16LE|UTF-32|UTF-32BE|UTF-32LE|
|--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |
|Smallest code point|0000|0000|0000|0000|0000|0000|0000
|Largest code point|10FFFF|10FFFF|10FFFF|10FFFF|10FFFF|10FFFF|10FFFF
|Code unit size|8 bits|16 bits|16 bits|16 bits|32 bits|32 bits|32 bits
|Byte order|N/A|`<BOM>`|big-endian|little-endian|`<BOM>`|big-endian|little-endian
|Fewest bytes per character|1|2|2|2|4|4|4
|Most bytes per character|4|4|4|4|4|4|4

## 历史原因

因为历史原因，曾经人们以为用两个8-bit，可以表示任意一字符，最初的Unicode标准就是16-bit的。所以Java中的char类型，C++中的wchar_t（gcc当作32-bit），QT中的QString，windows的底层Unicode的支持，都是16-bit的，所以造成了很多悲剧。

wchar_t实际上是个过时的东东，所以在C++11中增加了char16_t和char32_t类型，不过因为各家编译器的实现，标准库的实现，及语法等，实际使用还是相当相当的蛋疼。

这些悲剧一是unicode标准本身很比较晚才成熟，二则C/C++一直没有把unicode支持标准化（所以QT自己搞了一套，微软自己也搞了套）。

不过话说回来，这些悲剧不能全怪C++，只能说Unicode标准本身就蛋疼。据我所认识的编程语言中，只有后来比较晚出现的语言才比较好地支持unicode，比如golang，python3。

http://www.ruanyifeng.com/blog/2014/12/unicode.html     Unicode与JavaScript详解

## UTF-8

上面扯远了，再回来说下UTF-8。

### UTF-8的编码规则

UTF-8的编码规则可以看阮一锋写的文章：http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html

```
    Unicode符号范围 | UTF-8编码方式
(十六进制) | （二进制）
--------------------+---------------------------------------------
0000 0000-0000 007F | 0xxxxxxx
0000 0080-0000 07FF | 110xxxxx 10xxxxxx
0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
下面，还是以汉字“严”为例，演示如何实现UTF-8编码。
已知“严”的unicode是4E25（100111000100101），根据上表，可以发现4E25处在第三行的范围内（0000 0800-0000 FFFF），因此“严”的UTF-8编码需要三个字节，即格式是“1110xxxx 10xxxxxx 10xxxxxx”。然后，从“严”的最后一个二进制位开始，依次从后向前填入格式中的x，多出的位补0。这样就得到了，“严”的UTF-8编码是“11100100 10111000 10100101”，转换成十六进制就是E4B8A5。
```


要注意的是在UTF-8编码中，非ASCII字符的编码字串中不会出现0X00，即'\0'。这个很重要，是UTF-8有很多特性的重要原因。

### UTF-8编码的优点

UTF-8的编码规则，让它有很多特性：

* 兼容ASCII。ASCII编码的文件即同时也是UTF-8编码的文件。

* 可以从二进制数据流中识别出ASCII字符，比如有一个字节是0x7A，那么它肯定是字符'z'，因为它的最高为是0（参见上面的UTF-8的编码规则，所有的非ASCII字符的UTF-8编码的字节的最高位都是1）。

* 子串搜索可以直接按字节进行搜索，可以使用C语言原有的函数，如strchr，strstr（原因参考上面的UTF-8的编码规则）。

* 大多数处理8-bit编码文件的程序可以安全地处理utf-8文件（原因参考上一点）。

* utf-8编码的顺序和Unicode中码位(code point)的顺序是一致的，所以像unix工具，join，ls，sort等不用显式地指明是utf-8编码。

* utf-8编码是没有字节顺序的，UTF-8编码的文件可以不用写BOM头。

    像UTF-16或者UTF-32编码的文件就要写入BOM头，否则只能猜测了。（其实，文本编辑器都是读入一段数据，再猜测到底是什么编码。mozilla有个开源程序jChardet，可以猜什么编码，但是实测并不是很准确。对于没有指定编码的网页，浏览器只好去猜测所使用的编码，像chrome浏览器有时就会猜错，貌似错误率比IE要高。）


update:2016-05-23 
貌似这个库是一直更新的，可以用来检测编码，其它的都比较老了。

```xml
<dependency>
    <groupId>com.ibm.icu</groupId>
    <artifactId>icu4j</artifactId>
    <version>57.1</version>
</dependency>
```
http://stackoverflow.com/questions/499010/java-how-to-determine-the-correct-charset-encoding-of-a-stream

### utf-8编码的缺点

* utf-8编码是好，但是因为它是变长的！所以用一个什么类型来表示一个utf-8字符？

    这个在C++中还是无解，其实在其它的编程语言中同样是无解。至于go语言，它采取了一个折中的办法，在历遍字符串时，得到的是一个rune类型（实际即int32）。
    另外，假定有一个大文件，你修改了一个字符，那么有可能整一个文件都有重新保存。。
    UTF-16编码尽管也有可能要重新保存整个文件，但是概率比较小，因为大部分常用的字符都可以用一个16-bit来表示。

* 不能实现O(1)的随机访问

    所以像文本编辑器等要内部再用一个数组来记录每一个字符的位置。

不过，像随机访问，这样的操作是很少的。

对于编程语言来说，问题不大，因为大部分编程语言中string都是不要修改的（貌似只有C++中string是可以修改的）。

所以通常只有历遍操作，而历遍操作，对于UTF-8，UTF-16，UTF-32都是O(n)的时间复杂度（当然，如果较真，UTF-32编码要快一些）。

**其实上面的缺点，同样是UTF-16编码的缺点（它也是变长的1个16-bit或者2个16-bit），UTF-16编码还有一个重要的缺点是要指明字节序。**

UTF-32编码的缺点是要指明字节序和太浪费空间（如果全是ASCII字符，那么要浪费3倍的空间！），UTF-32编码的优点是可以O(1)实现随机访问。

## 其它的一些东东

受Windows API的影响，话说我以前是UTF-16党（wchar_t），但是后来发现它并不是定长的（不能用像s[i]这样的代码来访问一个字符），很伤心，慢慢改为用utf-8编码了，但是utf-8编码不能随机访问，的确也是个问题。

所以现在我是“无-党-派-人-士”:) 。

国外还有人专门做了个网站来推广utf-8编码：http://utf8everywhere.org/

还有专门吐槽UTF-16的：http://programmers.stackexchange.com/questions/102205/should-utf-16-be-considered-harmful

## Unicode在线查询的方法

* http://en.wikipedia.org/wiki/List_of_Unicode_characters

* http://www.unicode.org/charts/

4个byte的utf-8字符串的一些例子：

http://en.wikibooks.org/wiki/Unicode/Character_reference/1D000-1DFFF

这里还有一个测试utf-8解码是否正确的测试例子：

http://www.cl.cam.ac.uk/~mgk25/ucs/examples/UTF-8-test.txt

这里有一个Unicode编码清单，比较实用：

http://witmax.cn/unicode-list.html

## python3中的unicode

在python3中，字符串就是Unicode字符串。在CPython的代码中可以看到，对UTF-8，UTF-16，UTF-32都实现了支持。在创建一个字符串时（比如，解析“print('abc中国')”语句，在cmd窗口下，'abc中国'的编码通常是cp936，即gbk编码），如果是UTF-16或者UTF-32编码，则直接创建一个对应的字符串对象即可。如果是其它编码，则先转换为UTF-8编码，再创建一个字符串对象。

## java里如何处理大于U+FFFF（即4byte的UTF-16编码）的字符

java里用char来表示一个字符，但是char却不能表示大于U+FFFF的字符，因为char只有两个byte。上面说了java出现时unicode标准还没有成熟，所以这是一个历史遗留问题。

那么如何在java里表示和处理大于U+FFFF的字符？参考这里：

http://stackoverflow.com/questions/9834964/char-to-unicode-more-than-uffff-in-java

```java
// This represents U+10FFFF
String x = "\udbff\udfff";
```

或者

```java
String y = new StringBuilder().appendCodePoint(0x10ffff).toString();
```

还提供了一些函数来处理，更多的可以直接参考String类的注释。

```java
||System.out.println(y);
||System.out.println("codePoint:" + y.codePointAt(0));
||System.out.println("codePoint len:" + y.codePointCount(0, y.length()));
```

update: 2017-4-26

JDK里处理utf8编码的代码：http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/sun/nio/cs/UTF_8.java?av=f#285

JDK里默认的序列化是如何处理编码的：

`java.io.ObjectOutputStream.BlockDataOutputStream.writeUTFBody(String) `

因为jdk里的String都是utf-16的编码，在这里jdk的序列化做了一点压缩的处理。对于两个char（四个字节）的字符，压缩为3个字节来表示。

hessian里的字符是utf8编码的。参考 http://hessian.caucho.com/doc/hessian-serialization.html

在实现里，可以做一个优化。用unsafe 读出一个long，然后判断是不是全是ansi字符，这样子可以加快速度。

```java
      // fast path for ASCII
      int numLongs = toRead >> 3;
      for (int i =0; i < numLongs; i++) {
        long currentOffset = baseOffset + _offset;
        long test = unsafe.getLong(_buffer, currentOffset);
        if ((test & 0x8080808080808080L) == 0L) {
          _chunkLength-=8;
          toRead -= 8;
          for (int j = 0; j < 8; j++) {
            _sbuf.append((char)(_buffer[_offset++]));
          }
        } else {
          break;
        }
      }
```

## 总结

鉴于UTF-8编码有这么多优点，UTF-8编码会越来越流行。据google的统计数据，超过50%的网页，是utf-8编码。很多工具默认编码都是UTF-8的，比如python3的解析器，GCC。

通常来说UTF-8编码是优先选择，不过，如果是一些特殊应用，要用到O(1)的随机访问字符串，应该使用UTF-32编码。

## 参考

* UTF-8: Bits, Bytes, and Benefits http://research.swtch.com/utf8

* http://www.unicode.org

* UTF-8, UTF-16, UTF-32 & BOM http://www.unicode.org/faq/utf_bom.html

* 字符编码笔记：ASCII，Unicode和UTF-8 http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html 

* The Go Programming Language Specification http://golang.org/ref/spec

* 微软的Unicode的一个文档： http://msdn.microsoft.com/en-us/goglobal/bb688113.aspx  

* 其实你并不懂 Unicode： https://zhuanlan.zhihu.com/p/53714077 
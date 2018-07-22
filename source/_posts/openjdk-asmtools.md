---
title: OpenJDK里的AsmTools简介
date: 2018-07-12 21:03:54
tags:
 - jvm
 - java
 - asm

categories:
 - 技术
---

## 前言

* https://wiki.openjdk.java.net/display/CodeTools/asmtools 

在OpenJDK里有一个`AsmTools`项目，用来生成正确的或者不正确的java `.class`文件，主要用来测试和验证。

我们知道直接修改`.class`文件是很麻烦的，虽然有一些图形界面的工具，但还是很麻烦。

以前我的办法是用[ASMifier](https://static.javadoc.io/org.ow2.asm/asm/5.2/org/objectweb/asm/util/ASMifier.html)从`.class`文件生成asm java代码，再修改代码，生成新的`.class`文件，非常麻烦。

`AsmTools`引入了两种表示`.class`文件的语法：

* JASM

    用类似java本身的语法来定义类和函数，字节码指令则很像传统的汇编。

* JCOD 

    整个`.class`用容器的方式来表示，可以很清楚表示类文件的结构。

重要的是两种语法的文件都是可以和`.class`互相转换的。


## 构建AsmTools


官方文档： https://wiki.openjdk.java.net/display/CodeTools/How+to+build+AsmTools 

需要有jdk8和ant。

1. clone代码

    ```
    hg clone http://hg.openjdk.java.net/code-tools/asmtools
    ```
1. 编绎

    ```
    cd asmtools/build
    ant
    ```
    打包出来的zip包里有一个`asmtools.jar`。

也可以在这里下载我构建的：https://github.com/hengyunabc/hengyunabc.github.io/files/2188258/asmtools-7.0.zip

## 测试简单的java类



```java
public class Test {
	public static void main(String[] args) {
		System.out.println("hello");
	}
}
```

先用javac来编绎： 

```bash
javac Test.java
```

### 查看JASM语法结果

```bash
java -jar asmtools.jar jdis Test.class
```

结果：

```
super public class Test
	version 52:0
{


public Method "<init>":"()V"
	stack 1 locals 1
{
		aload_0;
		invokespecial	Method java/lang/Object."<init>":"()V";
		return;

}

public static Method main:"([Ljava/lang/String;)V"
	stack 2 locals 1
{
		getstatic	Field java/lang/System.out:"Ljava/io/PrintStream;";
		ldc	String "hello";
		invokevirtual	Method java/io/PrintStream.println:"(Ljava/lang/String;)V";
		return;

}

} // end Class Test
```

### 查看JCOD语法结果

```bash
java -jar asmtools.jar jdec Test.class
```

结果：

```
class Test {
  0xCAFEBABE;
  0; // minor version
  52; // version
  [] { // Constant Pool
    ; // first element is empty
    class #2; // #1
    Utf8 "Test"; // #2
    class #4; // #3
    Utf8 "java/lang/Object"; // #4
    Utf8 "<init>"; // #5
    Utf8 "()V"; // #6
    Utf8 "Code"; // #7
    Method #3 #9; // #8
    NameAndType #5 #6; // #9
    Utf8 "LineNumberTable"; // #10
    Utf8 "LocalVariableTable"; // #11
    Utf8 "this"; // #12
    Utf8 "LTest;"; // #13
    Utf8 "main"; // #14
    Utf8 "([Ljava/lang/String;)V"; // #15
    Field #17 #19; // #16
    class #18; // #17
    Utf8 "java/lang/System"; // #18
    NameAndType #20 #21; // #19
    Utf8 "out"; // #20
    Utf8 "Ljava/io/PrintStream;"; // #21
    String #23; // #22
    Utf8 "hello"; // #23
    Method #25 #27; // #24
    class #26; // #25
    Utf8 "java/io/PrintStream"; // #26
    NameAndType #28 #29; // #27
    Utf8 "println"; // #28
    Utf8 "(Ljava/lang/String;)V"; // #29
    Utf8 "args"; // #30
    Utf8 "[Ljava/lang/String;"; // #31
    Utf8 "SourceFile"; // #32
    Utf8 "Test.java"; // #33
  } // Constant Pool

  0x0021; // access
  #1;// this_cpx
  #3;// super_cpx

  [] { // Interfaces
  } // Interfaces

  [] { // fields
  } // fields

  [] { // methods
    { // Member
      0x0001; // access
      #5; // name_cpx
      #6; // sig_cpx
      [] { // Attributes
        Attr(#7) { // Code
          1; // max_stack
          1; // max_locals
          Bytes[]{
            0x2AB70008B1;
          }
          [] { // Traps
          } // end Traps
          [] { // Attributes
            Attr(#10) { // LineNumberTable
              [] { // LineNumberTable
                0  2;
              }
            } // end LineNumberTable
            ;
            Attr(#11) { // LocalVariableTable
              [] { // LocalVariableTable
                0 5 12 13 0;
              }
            } // end LocalVariableTable
          } // Attributes
        } // end Code
      } // Attributes
    } // Member
    ;
    { // Member
      0x0009; // access
      #14; // name_cpx
      #15; // sig_cpx
      [] { // Attributes
        Attr(#7) { // Code
          2; // max_stack
          1; // max_locals
          Bytes[]{
            0xB200101216B60018;
            0xB1;
          }
          [] { // Traps
          } // end Traps
          [] { // Attributes
            Attr(#10) { // LineNumberTable
              [] { // LineNumberTable
                0  5;
                8  6;
              }
            } // end LineNumberTable
            ;
            Attr(#11) { // LocalVariableTable
              [] { // LocalVariableTable
                0 9 30 31 0;
              }
            } // end LocalVariableTable
          } // Attributes
        } // end Code
      } // Attributes
    } // Member
  } // methods

  [] { // Attributes
    Attr(#32) { // SourceFile
      #33;
    } // end SourceFile
  } // Attributes
} // end class Test
```

## 从JASM/JCOD语法文件生成类文件

因为是等价表达，可以从JASM生成`.class`文件：

```bash
java -jar asmtools.jar jasm Test.jasm
```

同样可以从JCOD生成`.class`文件：

```bash
java -jar asmtools.jar jcoder Test.jasm
```

更多使用方法参考： https://wiki.openjdk.java.net/display/CodeTools/Chapter+2#Chapter2-Jasm.1

## 链接 

* https://wiki.openjdk.java.net/display/CodeTools/Appendix+A JASM Syntax

* https://wiki.openjdk.java.net/display/CodeTools/Appendix+B JCOD Syntax
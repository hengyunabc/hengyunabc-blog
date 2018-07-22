---
title: Java中异常Exception的实现的一些分析
date: 2013-11-03 16:58:28
tags:
 - java
 - exception
 - asm
 - bytecode


categories:
  - 技术

---

## 前言

最近发现一个很有用的Eclipse插件：http://andrei.gmxhome.de/bytecode/，可以在Eclipse直接查看，调试Java的字节码。

顺带研究了下Java里异常的实现机制，还有JDK7里的mutil catch的实现原理。



## athrow指令

在JVM里实现异常的指令是athrow，指令的参考在这里：http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.athrow

大段的英文就不粘贴过来了：）。

个人理解：JVM是基于所谓的栈帧的(stack frame)的，一个函数调用链就是一个个栈帧组成，当在一个栈里用athrow抛出异常时，JVM会搜索当前函数的异常处理表（参考下面的Class文件分析），如果有找到对应的异常处理的handler，则由这个handler来处理。如果没有，则清理当前栈，再回到上一层栈帧中处理。如果一层层栈帧回退，最终都没有找到Exception Handler，则线程终止。



下面贴点实际代码：

一个简单的函数：

```java
	public void testFunc(int i) throws NamingException, XPathException, SQLException {
		if (i == 3) {
			throw new XPathException("");
		}
		if (i == 4) {
			throw new SQLException();
		}
		if (i == 5) {
			throw new NamingException();
		}
	}

```

前面提到的ByteCode插件给出的分析：

```java
// access flags 0x1
  public testFunc(I)V throws javax/naming/NamingException javax/xml/xpath/XPathException java/sql/SQLException 
   L0
    LINENUMBER 31 L0
    ILOAD 1
    ICONST_3
    IF_ICMPNE L1
   L2
    LINENUMBER 32 L2
    NEW javax/xml/xpath/XPathException
    DUP
    LDC ""
    INVOKESPECIAL javax/xml/xpath/XPathException.<init>(Ljava/lang/String;)V
    ATHROW
   L1
    LINENUMBER 34 L1
   FRAME SAME
    ILOAD 1
    ICONST_4
    IF_ICMPNE L3
   L4
    LINENUMBER 35 L4
    NEW java/sql/SQLException
    DUP
    INVOKESPECIAL java/sql/SQLException.<init>()V
    ATHROW
   L3
    LINENUMBER 37 L3
   FRAME SAME
    ILOAD 1
    ICONST_5
    IF_ICMPNE L5
   L6
    LINENUMBER 38 L6
    NEW javax/naming/NamingException
    DUP
    INVOKESPECIAL javax/naming/NamingException.<init>()V
    ATHROW
   L5
    LINENUMBER 40 L5
   FRAME SAME
    RETURN
   L7
    LOCALVARIABLE this LTest; L0 L7 0
    LOCALVARIABLE i I L0 L7 1
    MAXSTACK = 3
    MAXLOCALS = 2

```

如果对汇编有一定了解的话，可以很容易看到，在Java里，抛出一个异常真的是非常简单的：
先New一个异常对象，再把这个对象的引用放到栈顶，再用athrow指令抛出这个异常。

## catch块的实现

那下面来看下，从指令层面，是如何处理这个异常的：

首先，处理这个异常的函数：

```java
	public void test() {
		try {
			testFunc(100);
		} catch (NamingException e) {
			// TODO Auto-generated catch block
			System.out.println("11111");
		} catch (XPathException e) {
			// TODO Auto-generated catch block
			System.out.println("22222");
		} catch (SQLException e) {
			// TODO Auto-generated catch block
			System.out.println("33333");
		}
	}
```

ByteCode插件给出的分析：

```java
 // access flags 0x1
  public test()V
    TRYCATCHBLOCK L0 L1 L2 javax/naming/NamingException
    TRYCATCHBLOCK L0 L1 L3 javax/xml/xpath/XPathException
    TRYCATCHBLOCK L0 L1 L4 java/sql/SQLException
   L0
    LINENUMBER 44 L0
    ALOAD 0
    BIPUSH 100
    INVOKEVIRTUAL Test.testFunc(I)V
   L1
    LINENUMBER 45 L1
    GOTO L5
   L2
   FRAME SAME1 javax/naming/NamingException
    ASTORE 1
   L6
    LINENUMBER 47 L6
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "11111"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L7
    GOTO L5
   L3
    LINENUMBER 48 L3
   FRAME SAME1 javax/xml/xpath/XPathException
    ASTORE 1
   L8
    LINENUMBER 50 L8
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "22222"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L9
    GOTO L5
   L4
    LINENUMBER 51 L4
   FRAME SAME1 java/sql/SQLException
    ASTORE 1
   L10
    LINENUMBER 53 L10
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "33333"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L5
    LINENUMBER 55 L5
   FRAME SAME
    RETURN
   L11
    LOCALVARIABLE this LTest; L0 L11 0
    LOCALVARIABLE e Ljavax/naming/NamingException; L6 L7 1
    LOCALVARIABLE e Ljavax/xml/xpath/XPathException; L8 L9 1
    LOCALVARIABLE e Ljava/sql/SQLException; L10 L5 1
    MAXSTACK = 2
    MAXLOCALS = 2
```

可以看到最开始部分有三条TRYCATCHBLOCK，再分析下这些TRYCATCHBLOCK后面跟着三个标签，最后还有一个异常的名字，再仔细分析下，可以发现三个标签分别对应try块开始的地方，try块结束的地方，catch块开始的地方。这个实际上就是所谓的Execption Table。

## class文件格式分析


另外，在Class文件的格式里，我们也可以看到Method的Execption Table。可以看出一个条目有四个元素组成：

start_pc, end_pc, handler_pc, catch_type。显然这些异常表里的数据是和代码位置有关的，和我们上面看到的一致。

```java
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

所以，我们可以看到，test()函数调用了testFunc()函数，那么，当testFunc()函数里抛出异常时，JVM先回退到test()函数的栈帧，再从Execption Table里查找是否有合适的Execption Hanler，查找首先当前的pc(program counter)要在start_pc, end_pc之间，而且异常的名字要匹配（当然这个应该会被优化成常量的比较，即一个long的比较，不会真的去比较字符串）。如果找到，则跳到对应的handler_pc处继续执行。

## finally块的实现

下面再来看下Finally块到底是怎么实现的：

在代码里增加finally块：

```java
	public void test2() {
		try {
			testFunc(100);
		} catch (NamingException e) {
			// TODO Auto-generated catch block
			System.out.println("11111");
		} catch (XPathException e) {
			// TODO Auto-generated catch block
			System.out.println("22222");
		} catch (SQLException e) {
			// TODO Auto-generated catch block
			System.out.println("33333");
		}finally {
			System.out.println("xxxxxxxxxxxx");
		}
	}
```

ByteCode插件的分析：

```java
// access flags 0x1
  public test2()V
    TRYCATCHBLOCK L0 L1 L2 javax/naming/NamingException
    TRYCATCHBLOCK L0 L1 L3 javax/xml/xpath/XPathException
    TRYCATCHBLOCK L0 L1 L4 java/sql/SQLException
    TRYCATCHBLOCK L0 L5 L6 
    TRYCATCHBLOCK L3 L7 L6 
    TRYCATCHBLOCK L4 L8 L6 
   L0
    LINENUMBER 60 L0
    ALOAD 0
    BIPUSH 100
    INVOKEVIRTUAL Test.testFunc(I)V
   L1
    LINENUMBER 61 L1
    GOTO L9
   L2
   FRAME SAME1 javax/naming/NamingException
    ASTORE 1
   L10
    LINENUMBER 63 L10
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "11111"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L5
    LINENUMBER 71 L5
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "xxxxxxxxxxxx"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
    GOTO L11
   L3
    LINENUMBER 64 L3
   FRAME SAME1 javax/xml/xpath/XPathException
    ASTORE 1
   L12
    LINENUMBER 66 L12
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "22222"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L7
    LINENUMBER 71 L7
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "xxxxxxxxxxxx"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
    GOTO L11
   L4
    LINENUMBER 67 L4
   FRAME SAME1 java/sql/SQLException
    ASTORE 1
   L13
    LINENUMBER 69 L13
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "33333"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L8
    LINENUMBER 71 L8
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "xxxxxxxxxxxx"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
    GOTO L11
   L6
    LINENUMBER 70 L6
   FRAME SAME1 java/lang/Throwable
    ASTORE 2
   L14
    LINENUMBER 71 L14
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "xxxxxxxxxxxx"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L15
    LINENUMBER 72 L15
    ALOAD 2
    ATHROW
   L9
    LINENUMBER 71 L9
   FRAME SAME
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "xxxxxxxxxxxx"
    INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
   L11
    LINENUMBER 73 L11
   FRAME SAME
    RETURN
   L16
    LOCALVARIABLE this LTest; L0 L16 0
    LOCALVARIABLE e Ljavax/naming/NamingException; L10 L5 1
    LOCALVARIABLE e Ljavax/xml/xpath/XPathException; L12 L7 1
    LOCALVARIABLE e Ljava/sql/SQLException; L13 L8 1
    MAXSTACK = 2
    MAXLOCALS = 3
```

我们可以很神奇的发现，finally块的代码在每一个catch后面都有一份。也就是说finally的实现有点像内联优化，把代码复制了很多份。

## JDK7中mutil catch的实现

我们再来看下JDK7新增的mutil catch语法的实现：

```java
	public void test3() {
		try {
			testFunc(100);
		} catch (NamingException | XPathException | SQLException e) {
			e.printStackTrace();
		}
	}

```

ByteCode插件的分析：

```java
 public test3()V
    TRYCATCHBLOCK L0 L1 L2 javax/naming/NamingException
    TRYCATCHBLOCK L0 L1 L2 javax/xml/xpath/XPathException
    TRYCATCHBLOCK L0 L1 L2 java/sql/SQLException
   L0
    LINENUMBER 77 L0
    ALOAD 0
    BIPUSH 100
    INVOKEVIRTUAL Test.testFunc(I)V
   L1
    LINENUMBER 78 L1
    GOTO L3
   L2
   FRAME SAME1 java/lang/Exception
    ASTORE 1
   L4
    LINENUMBER 79 L4
    ALOAD 1
    INVOKEVIRTUAL java/lang/Exception.printStackTrace()V
   L3
    LINENUMBER 81 L3
   FRAME SAME
    RETURN
   L5
    LOCALVARIABLE this LTest; L0 L5 0
    LOCALVARIABLE e Ljava/lang/Exception; L4 L3 1
    MAXSTACK = 2
    MAXLOCALS = 2
```


我们可以发现，每一个TRYCATCHBLOCK的配置都是一样的，只是异常的名字不一样。所以实际上mutil catch的实现和普通的实现没有太大的区别，当然从JVM的实现角度来看，mutil catch有可能可以优化Exception Handler的查找过程（纯猜测的，如果是线性查找，则效率是一样的）。不过有好处是可以减少class文件的体积，这个也比较有用，因为目前Java的class文件的大小是有限制的。参考这里；http://stackoverflow.com/questions/5497495/maximum-size-of-java-class-exception-table

## 总结

Java中的异常的实现不是什么太神秘的东东，和人们的直觉的实现差不多。任何编程语言的异常机制都会有一定的开销，但是异常如果没有触发，实际上是没有开销的。

异常在触发时，要new一个异常对象，再一层层地栈帧回退，每层都要查找异常处理表，开销还是比较大的。

所在异常只应该用在合适的地方，如果异常像Switch那样用，那就悲剧了。

## 参考

* http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.athrow

* http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.10

* http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.3

* http://andrei.gmxhome.de/bytecode/    非常有用的分析Java 汇编代码的Eclipse插件
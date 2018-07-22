---
title: Kryo简介及代码阅读笔记
date: 2012-07-19 19:58:28
tags:
 - java
 - kryo
 - serialization


categories:
  - 技术

---

## 更新

更新：2012-08-01

版本 2.16长时间运行可能会导致OOM，版本2.18有bug，不能正确序列化map和collection。

真是悲剧，所用的每一个版本都有bug。不过从代码来看，作者有时的确比较随便。。测试用例也少。。（比起msgpack少多了）

## 简介

Kryo官方网站：https://code.google.com/p/kryo/

### 优点

* 速度快！见https://github.com/eishay/jvm-serializers/wiki/Staging-Results

* 支持互相引用，比如类A引用类B，类B引用类A，可以正确地反序列化。

* 支持多个引用同一个对象，比如多个类引用了同一个对象O，只会保存一份O的数据。

* 支持一些有用的注解，如@Tag，@Optional。

* 支持忽略指定的字段。

* 支持null

* 代码入侵少

* 代码比较简法（比起msgpack，少得多）


### 缺点

* bug多 2.12，2.14都有bug

* 文档比较少，有些使用方法要看代码才能理解，最新版2.14有bug，不能正确反序列化map类型。

* 不是跨语言的解决方案

* 貌似每一个类都要注册下，不然只能用writeClassAndObject和readClassAndObject函数。

* 类要有一个空的构造函数，不然不能序列化。

* （如果这个构造函数里调用了别的资源，而这个资源没有初始化，那么就悲剧了。）

* ~~可以通过实现KryoSerializable接口来避免这个问题。。~~ 同样不能解决这个问题

   Java自带的则不用调用这个构造函数。

* msgpack同样有这个问题。

### 接口

* KryoSerializable

* KryoCopyable

### 实现忽略指定的字段

1. 使用transient关键字

1. 使用Context结合@Optional注解

1. 使用@Tag注解（很麻烦）

1. 实现KryoSerializable接口（比较麻烦，相当于手写代码）

### 引用

可以用setReferences(boolean )函数来设置，默认是打开的。

引用的实现原理：

原本以为要实现引用是个比较麻烦的事，因为一想到引用，头脑中就出现一个图。。但在看了代码后，发现是比较简单的。

在Kryo类中有一个writtenObjects的ArrayList，记录已写入的对象。有一个readObjects来记录已写入的对象。

另外有个depth来记录深度，每写一个对象时depth++，当写完时depth--，当depth == 0时，调用reset函数，清空writtenObjects和。

比如写一个大对象，这个对象有很多成员，每一个成员都是一个对象，而成员之间有可能用引用关系（A引用了B，B也引用了A）。

```java
private int depth, maxDepth = Integer.MAX_VALUE, nextRegisterID;
private final ArrayList writtenObjects = new ArrayList();
private final ArrayList readObjects = new ArrayList();
```

每当写一个对象时，都到里面去检查有没有这个对象，如果有的话，就只写一个int即可。这个int是要表明这个对象当前在的位置即可。

因为当反序列化时，可以据读到的int，正确地从readObjects取回对象。

如果没有，则在输出流中写入writtenObjects的size()+1，再把这个对象放到writtenObjects中。

序列化时，写入引用的对象在writtenObjects中的位置：

```java
    for (int i = 0, n = writtenObjects.size(); i < n; i++) {
    if (writtenObjects.get(i) == object) {
        if (DEBUG)
            debug("kryo", "Write object reference " + i + ": "
                    + string(object));
        output.writeInt(i + 1, true); // + 1 because 0 means null.
        return true;
    }
}
// Only write the object the first time encountered in object graph.
output.writeInt(writtenObjects.size() + 1, true);
writtenObjects.add(object);
```

反序列化时，据id从readObjects得到正确的对象：

```java
    if (--id < readObjects.size()) {
    Object object = readObjects.get(id);
    if (DEBUG)
        debug("kryo", "Read object reference " + id + ": "
                + string(object));
    return object;
}
```

### 注册

貌似每一个类都要注册下，不然只能用writeClassAndObject和readClassAndObject函数。

注册的顺序不能乱！！因为是每一个类都有一个id，而这个id是增长的！

可以设置registrationRequired，防止没有注册的情况！

### 注解annotation

貌似只有四个：@Optional，@Tag，@DefaultSerializer，@NotNull

### 实现原理

每注册一个类，都有一个id，由一个IntMap的hashMap来保存（TODO，研究下这个东东的实现）

### 代码阅读笔记

在Kryo类中有以下的成员，简单来看，就是一些HashMap，用来存放id和Class，Class和id，id和Registration，Class和Registration之间的对应关系：

```java
private final IntMap<Registration> idToRegistration = new IntMap();
private final ObjectMap<Class, Registration> classToRegistration = new ObjectMap();
private final IdentityObjectIntMap<Class> classToNameId = new IdentityObjectIntMap();
private final IntMap<Class> nameIdToClass = new IntMap();
private int nextNameId;  //不断增长，分新的Class分配一个新的id，即Registration中的id

```

每一个类对应一个Registration：

```java
    public class Registration {
    private final Class type;
    private final int id;         //这个要注意，每一个类都有一个唯一的id，这个id是从0开始不断增长的
    private Serializer serializer;
    private ObjectInstantiator instantiator;  //复制对象时候用
}
```

直接看这个类的成员，就大概能明白它是怎样回事了。

要注意一点，Registration中的id很重要，可以说是和别的序列化方案相比，高效之处。

在调用Kryo.writeClass(Output output, Class type)函数时，

先查找到这个类的Registration，得到Serializer，再调用write (Kryo kryo, Output output, T object)写到输出流中。



如果没有找到的话，则为这个类生成一个Registration，并放到Kryo类中的对应的HashMap中。

再来说下Serializer：

默认是FieldSerializer，在生成Registration中，如果为这个类找不到Serializer（到defaultSerializers中找），

则会构造一个FieldSerializer。

FieldSerializer实际是有一个数组存放了每一个field的信息，当调用write (Kryo kryo, Output output, T object)函数时，则历遍所有的field，把每一个field写到输出流中。

```java
    private CachedField[] fields = new CachedField[0];
                                                     
public class CachedField<X> {
    final Field field;
    Class fieldClass;
    Serializer serializer;
    boolean canBeNull;
    int accessIndex = -1;
｝
```

Kryo有两种模式，一种是先注册(regist)，再写对象，即writeObject函数，实际上如果不先注册，在写对象时也会注册，并为class分配一个id。

注意，如果是rpc，则必须两端都按同样的顺序注册，否则会出错，因为必须要明确类对应的唯一id。



另一种是写类名及对象，即writeClassAndObject函数。

writeClassAndObject函数是先写入(-1 + 2)（一个约定的数字），再写入类ID（第一次要先写-1，再写类ID + 类名），写入引用关系（见引用的实现），最后才写真正的数据）。

注意每一次writeClassAndObject调用后信息都会清空，所以不用担心和client交互时会出错。

### 代码中其它有意思的地方

在writeString函数中先判断是不是ascii即，所有都<127，如果是在写string的最后一个字符，会执行这个：

```java
    buffer[position - 1] |= 0x80;
```

不知为何。。

在readString的时候也会判断：

```java
    if ((b & 0x80) == 0) {
// ascii
```

是因为这个http://topic.csdn.net/t/20040416/10/2972001.html ？所谓的双字节的？

### 为什么比其它的序列化方案要快？

为每一个类分配一个id

实现了自己的IntMap

代码中一些取巧的地方：

利用变量memoizedRegistration和memoizedType记录上一次的调用writeObject函数的Class，则如果两次写入同一类型时，可以直接拿到，不再查找HashMap。

这个也许是为什么在测试中kryo要比其它类库要快的原因之一。

### 注意事项

实现KryoSerializable接口

像下面这样实现是错误的。

```java
@Override
public void write(Kryo kryo, Output output) {
    // TODO Auto-generated method stub
    kryo.writeObject(output, this);
}
                             
@Override
public void read(Kryo kryo, Input input) {
    // TODO Auto-generated method stub
    kryo.readObject(input, this.getClass());
}
```

实际上只写入了默认构造函数的内容！

原因是在生成Registration时已在writtenObjects中写入了这个类，所以在Kryo类中的writtenObjects中已有这个类，所以在调用write函数时，如果是用下面的代码，则会以为这个类是已写过的，所以直接写了一个1和它的id！！

实际上如果实现了KryoSerializable接口，最终是这个类来调用接口的write函数：KryoSerializableSerializer

**正确的写法是写入每一个成员，在read函数中把数据读出，再赋值给成员。**
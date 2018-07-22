---
title: 详细分析罕见的ClassCircularityError异常导致的StackOverflowError
date: 2016-04-10 21:58:28
tags:
 - java
 - jvm


categories:
  - 技术

---

先打一个广告。

greys是一个很不错的java诊断工具：https://github.com/oldmanpushcart/greys-anatomy

---

最近尝试用greys来实时统计jvm里的异常生成数量，在增强Throwable时，发现应用会抛出StackOverflowError。下面记录详细的分析过程。

在真正分析之前，先介绍`JVM对反射方法调用的优化`和`greys的工作原理`。

## JVM对反射方法调用的优化
在JVM里对于反射方法调用`Method.invoke`，默认情况下，是通过NativeMethodAccessorImpl来调用到的。

调用栈如下：

```java
	NativeMethodAccessorImpl.invoke0(Method, Object, Object[]) line: not available [native method]	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 62	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
```

当经过16次方法调用之后，NativeMethodAccessorImpl 会用MethodAccessorGenerator 动态生成一个MethodAccessorImpl（即下面的GeneratedMethodAccessor1） ，然后再设置到 DelegatingMethodAccessorImpl 里。然后调用栈就变成这个样子：

```java
	GeneratedMethodAccessor1.invoke(Object, Object[]) line: not available	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
```

这个动态生成的GeneratedMethodAccessor1是如何加载到ClassLoader里的？实际上是通过 `Unsafe.defineClass` 来define，然后再调用 `ClassLoader.loadClass(String)` 来加载到的。

```java
    AgentLauncher$1(ClassLoader).loadClass(String) line: 357	
    Unsafe.defineClass(String, byte[], int, int, ClassLoader, ProtectionDomain) line: not available [native method] 
    ClassDefiner.defineClass(String, byte[], int, int, ClassLoader) line: 63  
```

更多反射调用优化的细节参考：http://rednaxelafx.iteye.com/blog/548536

简单总结下：

* jvm会对method反射调用优化
* 运行时动态生成反射调用代码，再define到classloader里
* define到classloader时，会调用`ClassLoader.loadClass(String)`

## greys的工作原理

使用greys可以在运行时，对方法调用进行一些watch, monitor等的动作。那么这个是怎么实现的呢？

简单来说，是通过运行时修改字节码来实现的。比如下面这个函数：

```java
class xxx {
    public String abc(Student s) {
        return s.getName();
    }
}
```

被greys修改过后，变为

```java
                Spy.ON_BEFORE_METHOD.invoke(null, new Integer(0), xxx2.getClass().getClassLoader(), "xxx", "abc", "(LStudent;)Ljava/lang/String;", xxx2, {student});
                try {
                    void s;
                    String string = s.getName();
                    Spy.ON_RETURN_METHOD.invoke(null, string);
                    return string;
                }
                catch (Throwable v1) {
                    Spy.ON_THROWS_METHOD.invoke(null, v1);
                    throw v1;
                }
```

可以看到，greys在原来的method里插入很多钩子，所以greys可以获取到method被调用的参数，返回值等信息。

## 当使用greys对java.lang.Throwable来增强时，会抛出StackOverflowError

测试代码：
```
public class ExceptionTest {
	public static void main(String[] args) throws Exception {
		for (int i = 0; i < 100000; i++) {
			RuntimeException exception = new RuntimeException("");
			System.err.println(exception);

			Thread.sleep(1000);
		}
	}
}
```

在命令行里attach到测试代码进程之后，在greys console里执行

```bash
options unsafe true
monitor -c 1 java.lang.Throwable *
```

当用greys增强java.lang.Throwable之后，经过16秒之后，就会抛出StackOverflowError。

具体的异常栈很长，这里只贴出重点部分：

```java
Thread [main] (Suspended (exception StackOverflowError))	
	ClassLoader.checkCreateClassLoader() line: 272	
...
	ClassCircularityError(Throwable).<init>(String) line: 264	
	ClassCircularityError(Error).<init>(String) line: 70	
	ClassCircularityError(LinkageError).<init>(String) line: 55	
	ClassCircularityError.<init>(String) line: 53	
	Unsafe.defineClass(String, byte[], int, int, ClassLoader, ProtectionDomain) line: not available [native method]	
	ClassDefiner.defineClass(String, byte[], int, int, ClassLoader) line: 63	
	MethodAccessorGenerator$1.run() line: 399	
	MethodAccessorGenerator$1.run() line: 394	
	AccessController.doPrivileged(PrivilegedAction<T>) line: not available [native method]	
	MethodAccessorGenerator.generate(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int, boolean, boolean, Class<?>) line: 393	
	MethodAccessorGenerator.generateMethod(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int) line: 75	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 53	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
	ClassCircularityError(Throwable).<init>(String) line: 264	
	ClassCircularityError(Error).<init>(String) line: 70	
	ClassCircularityError(LinkageError).<init>(String) line: 55	
	ClassCircularityError.<init>(String) line: 53	
	Unsafe.defineClass(String, byte[], int, int, ClassLoader, ProtectionDomain) line: not available [native method]	
	ClassDefiner.defineClass(String, byte[], int, int, ClassLoader) line: 63	
	MethodAccessorGenerator$1.run() line: 399	
	MethodAccessorGenerator$1.run() line: 394	
	AccessController.doPrivileged(PrivilegedAction<T>) line: not available [native method]	
	MethodAccessorGenerator.generate(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int, boolean, boolean, Class<?>) line: 393	
	MethodAccessorGenerator.generateMethod(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int) line: 75	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 53	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
	ClassNotFoundException(Throwable).<init>(String, Throwable) line: 286	
	ClassNotFoundException(Exception).<init>(String, Throwable) line: 84	
	ClassNotFoundException(ReflectiveOperationException).<init>(String, Throwable) line: 75	
	ClassNotFoundException.<init>(String) line: 82	
	AgentLauncher$1(URLClassLoader).findClass(String) line: 381	
	AgentLauncher$1.loadClass(String, boolean) line: 55	
	AgentLauncher$1(ClassLoader).loadClass(String) line: 357	
	Unsafe.defineClass(String, byte[], int, int, ClassLoader, ProtectionDomain) line: not available [native method]	
	ClassDefiner.defineClass(String, byte[], int, int, ClassLoader) line: 63	
	MethodAccessorGenerator$1.run() line: 399	
	MethodAccessorGenerator$1.run() line: 394	
	AccessController.doPrivileged(PrivilegedAction<T>) line: not available [native method]	
	MethodAccessorGenerator.generate(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int, boolean, boolean, Class<?>) line: 393	
	MethodAccessorGenerator.generateMethod(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int) line: 75	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 53	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
	RuntimeException(Throwable).<init>(String) line: 264	
	RuntimeException(Exception).<init>(String) line: 66	
	RuntimeException.<init>(String) line: 62	
	ExceptionTest.main(String[]) line: 15	
```

从异常栈可以看出，先出现了一个ClassNotFoundException，然后大量的ClassCircularityError，最终导致StackOverflowError。

下面具体分析原因。

## 被增强过后的Throwable的代码

当`monitor -c 1 java.lang.Throwable * `命令执行之后，Throwable的代码实际上变为这个样子：

```java
public class Throwable {
    public Throwable() {
        Spy.ON_BEFORE_METHOD.invoke(...);
        try {
            // Throwable <init>
        }
        catch (Throwable v1) {
            Spy.ON_THROWS_METHOD.invoke(null, v1);
            throw v1;
        }
    }
}
```

这个`Spy.ON_BEFORE_METHOD.invoke` 是一个反射调用，那么当它被调用16次之后，jvm会生成优化的代码。从最开始的异常栈可以看到这些信息：

```java
	Unsafe.defineClass(String, byte[], int, int, ClassLoader, ProtectionDomain) line: not available [native method]	
	ClassDefiner.defineClass(String, byte[], int, int, ClassLoader) line: 63	
	MethodAccessorGenerator$1.run() line: 399	
	MethodAccessorGenerator$1.run() line: 394	
	AccessController.doPrivileged(PrivilegedAction<T>) line: not available [native method]	
	MethodAccessorGenerator.generate(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int, boolean, boolean, Class<?>) line: 393	
	MethodAccessorGenerator.generateMethod(Class<?>, String, Class<?>[], Class<?>, Class<?>[], int) line: 75	
	NativeMethodAccessorImpl.invoke(Object, Object[]) line: 53	
	DelegatingMethodAccessorImpl.invoke(Object, Object[]) line: 43	
	Method.invoke(Object, Object...) line: 497	
	RuntimeException(Throwable).<init>(String) line: 264	
	RuntimeException(Exception).<init>(String) line: 66	
	RuntimeException.<init>(String) line: 62	
	ExceptionTest.main(String[]) line: 15	
```

这时，生成的反射调用优化类名字是`sun/reflect/GeneratedMethodAccessor1`。

## ClassNotFoundException 怎么产生的

接着，代码抛出了一个ClassNotFoundException，这个ClassNotFoundException来自`AgentLauncher$1(URLClassLoader)`。这是AgentLauncher 里自定义的一个URLClassLoader。

这个自定义ClassLoader的逻辑很简单，优先从自己查找class，如果找不到则从parent里查找。这是一个常见的重写ClassLoader的逻辑。

```java
            classLoader = new URLClassLoader(new URL[]{new URL("file:" + agentJar)}) {

                @Override
                protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
                    final Class<?> loadedClass = findLoadedClass(name);
                    if (loadedClass != null) {
                        return loadedClass;
                    }

                    try {
                        Class<?> aClass = findClass(name);
                        if (resolve) {
                            resolveClass(aClass);
                        }
                        return aClass;
                    } catch (Exception e) {
                        return super.loadClass(name, resolve);
                    }
                }

            };
```

这个ClassNotFoundException的具体信息是`sun.reflect.MethodAccessorImpl`。实际上是MethodAccessorGenerator在生成反射调用代码里要用到的，所以需要加载到ClassLoader里。因此自定义的URLClassLoader在findClass时抛出了一个ClassNotFoundException。

## ClassCircularityError是怎么产生的

抛出的ClassNotFoundException是Throwable的一个子类，所以也会调用Throwable的构造函数，然后需要调用到`Spy.ON_BEFORE_METHOD.invoke` 。

注意，这时`Spy.ON_BEFORE_METHOD.invoke`的反射调用代码已经生成了，但是还没有置入到ClassLoader里，也没有放到DelegatingMethodAccessorImpl里。所以这时仍然调用的是NativeMethodAccessorImpl，然后再次生成反射调用类，name是`sun/reflect/GeneratedMethodAccessor2`。

生成`GeneratedMethodAccessor2`之后， 会调用`Unsafe.define`来define这个class。这里抛出了ClassCircularityError。

## 为什么会抛出ClassCircularityError

因为`Unsafe.defineClass` 是native实现，所以需要查看hotspot源码才能知道具体的细节。

SystemDictionary是jvm里加载的所有类的总管，所以在defineClass，会调用到这个函数

```cpp
// systemDictionary.cpp
Klass* SystemDictionary::resolve_instance_class_or_null(Symbol* name,
                                                        Handle class_loader,
                                                        Handle protection_domain,
                                                        TRAPS) {
```

然后，在这里面会有一个判断循环的方法。防止循环依赖。如果是发现了循环，则会抛出ClassCircularityError。

```cpp
// systemDictionary.cpp
          // only need check_seen_thread once, not on each loop
          // 6341374 java/lang/Instrument with -Xcomp
          if (oldprobe->check_seen_thread(THREAD, PlaceholderTable::LOAD_INSTANCE)) {
            throw_circularity_error = true;
          }
...

  if (throw_circularity_error) {
      ResourceMark rm(THREAD);
      THROW_MSG_NULL(vmSymbols::java_lang_ClassCircularityError(), child_name->as_C_string());
  }
```

这个循环检测是怎么工作的呢？

实际上是把线程放到一个queue里，然后判断这个queue里的保存的前一个线程是不是一样的，如果是一样的，则会认为出现循环了。

```cpp
// placeholders.cpp
  bool check_seen_thread(Thread* thread, PlaceholderTable::classloadAction action) {
    assert_lock_strong(SystemDictionary_lock);
    SeenThread* threadQ = actionToQueue(action);
    SeenThread* seen = threadQ;
    while (seen) {
      if (thread == seen->thread()) {
        return true;
      }
      seen = seen->next();
    }
    return false;
  }

  SeenThread* actionToQueue(PlaceholderTable::classloadAction action) {
    SeenThread* queuehead;
    switch (action) {
      case PlaceholderTable::LOAD_INSTANCE:
         queuehead = _loadInstanceThreadQ;
         break;
      case PlaceholderTable::LOAD_SUPER:
         queuehead = _superThreadQ;
         break;
      case PlaceholderTable::DEFINE_CLASS:
         queuehead = _defineThreadQ;
         break;
      default: Unimplemented();
    }
    return queuehead;
  }
```

就这个例子实际情况来说，就是同一个thread里，在defineClass时，再次defineClass，这样子就出现了循环。所以抛出了一个ClassCircularityError。

## StackOverflowError怎么产生的

OK，搞明白ClassCircularityError这个异常是怎么产生的之后，回到原来的流程看下。

这个ClassCircularityError也是Throwable的一个子类，那么它也需要初始化，然后调用`Spy.ON_BEFORE_METHOD.invoke` ……

然后，接下来就生成一个`sun/reflect/GeneratedMethodAccessor3` ，然后会被defindClass，然后就会检测到循环，然后再次抛出ClassCircularityError。 

就这样子，最终一直到StackOverflowError

## 完整的异常产生流程

* Throwable的构造函数被增强之后，需要调用`Spy.ON_BEFORE_METHOD.invoke`
* `Spy.ON_BEFORE_METHOD.invoke`经过16次调用之后，jvm会生成反射调用优化代码
* 反射调用优化类`sun/reflect/GeneratedMethodAccessor1`需要被自定义的ClassLoader加载
* 自定义的ClassLoader重写了loadClass函数，抛出了一个ClassNotFoundException
* ClassNotFoundException在构造时，调用了Throwable的构造函数，然后调用了`Spy.ON_BEFORE_METHOD.invoke`
* `Spy.ON_BEFORE_METHOD.invoke` 生成反射调用优化代码：`sun/reflect/GeneratedMethodAccessor2`
* Unsafe在defineClass `sun/reflect/GeneratedMethodAccessor2` 时，检测到循环，抛出了ClassCircularityError
* ClassCircularityError在构造时，调用了Throwable的构造函数，然后调用了`Spy.ON_BEFORE_METHOD.invoke`
* 反射调用优化类`sun/reflect/GeneratedMethodAccessor3` 在defineClass时，检测到循环，抛出了ClassCircularityError

* …… 不断抛出ClassCircularityError，最终导致StackOverflowError

## 总结

这个问题的根源是在Throwable的构造函数里抛出了异常，这样子明显无解。

为了避免这个问题，需要保证增强过后的Throwable的构造函数里不能抛出任何异常。然而因为jvm的反射调用优化，导致ClassLoader在loadClass时抛出了异常。所以要避免在加载jvm生成反射优化类时抛出异常。

修改过后的自定义URLClassLoader代码：

```java
classLoader = new URLClassLoader(new URL[]{new URL("file:" + agentJar)}) {
    @Override
    protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        final Class<?> loadedClass = findLoadedClass(name);
        if (loadedClass != null) {
            return loadedClass;
        }
        
        // 优先从parent（SystemClassLoader）里加载系统类，避免抛出ClassNotFoundException
        if(name != null && (name.startsWith("sun.") || name.startsWith("java."))) {
          return super.loadClass(name, resolve);
        }

        try {
            Class<?> aClass = findClass(name);
            if (resolve) {
                resolveClass(aClass);
            }
            return aClass;
        } catch (Exception e) {
          // ignore
        }
        return super.loadClass(name, resolve);
    }
};
```

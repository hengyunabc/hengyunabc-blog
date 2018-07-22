---
title: 详细解析boost中bind的实现
date: 2012-07-26 21:58:28
tags:
 - cpp
 - boost


categories:
  - 技术

---


## 写在前面的话

在C++11之后，std::bind是C++标准库的一个组件了。一开始想弄个C++11的实现来研究下，发现里面用到了可变参数模板（代码变得非常神奇）.

http://llvm.org/svn/llvm-project/libcxx/trunk/include/functional

还是弄个原始点的boost的实现来研究下。

话说网上关于boost::bind的实现的文章也有不少，不过大多数都是贴一段代码，再扯一通，结果到头来什么都没看明白。（起码LZ是。。）

花了一天的功夫，最终从boost::bind的源代码中抠出了可编绎运行的代码。

下面一步一步来解析boost::bind的实现。

## 标准库中的fouctor和bind1st的实现

首先从简单的入手。先看下标准库中的fouctor和bind1st的实现（为了防止和标准库的命名冲突，全部都在命名后面加了2）。

```cpp
#include <iostream>
#include <algorithm>
using namespace std;
template<typename _Arg1, typename _Arg2, typename _Result>
struct binary_function2 {
	typedef _Arg1 first_argument_type;
	typedef _Arg2 second_argument_type;
	typedef _Result result_type;
};
 
template<typename _Tp>
struct equal_to2: public binary_function2<_Tp, _Tp, bool> {
	bool operator()(const _Tp& __x, const _Tp& __y) const {
		return __x == __y;
	}
};
 
template<typename _Arg, typename _Result>
struct unary_function2 {
	typedef _Arg argument_type;
	typedef _Result result_type;
};
 
template<typename _Operation>
class binder1st2: public unary_function2<
		typename _Operation::second_argument_type,
		typename _Operation::result_type> {
protected:
	_Operation op;
	typename _Operation::first_argument_type value;
 
public:
	binder1st2(const _Operation& __x,
			const typename _Operation::first_argument_type& __y) :
			op(__x), value(__y) {
	}
 
	typename _Operation::result_type operator()(
			const typename _Operation::second_argument_type& __x) const {
		return op(value, __x);
	}
 
	typename _Operation::result_type operator()(
			typename _Operation::second_argument_type& __x) const {
		return op(value, __x);
	}
};
 
template<typename _Operation, typename _Tp>
inline binder1st2<_Operation> bind1st2(const _Operation& __fn, const _Tp& __x) {
	typedef typename _Operation::first_argument_type _Arg1_type;
	return binder1st2<_Operation>(__fn, _Arg1_type(__x));
}
 
int main() {
	binder1st2<equal_to2<int> > equal_to_10(equal_to2<int>(), 10);
	int numbers[] = { 10, 20, 30, 40, 50, 10 };
	int cx;
	cx = std::count_if(numbers, numbers + 6, bind1st(equal_to2<int>(), 10));
	cout << "There are " << cx << " elements that are equal to 10.\n";
}
```

上面的实现还是比较简单的，希望还没有看晕。:)

从代码可以看出binder1st的实现，实制上是把参数保存起来，等到调用时，再取出来使用。

boost::bind从原理上来说，也是这么回事，但是要复杂得多。

## 解析简化版bind2

下面是从boost::bind源码中抠出来的，一个简单的bind的实现（bind改为bind2），主流编译器应该都可以编统执行。

为了方便理解程序运行过程，代码中增加了一些cout输出。

myBind.h：

```cpp
#ifndef BOOST_BIND_BIND_HPP_INCLUDED__Mybind
#define BOOST_BIND_BIND_HPP_INCLUDED__Mybind
 
#include <iostream>
using namespace std;
namespace boost {
template<class T> struct is_placeholder {
	enum _vt {
		value = 0
	};
};
template<int I> struct arg {
	arg() {
	}
};
template<class T>
struct type {
};
namespace _bi // implementation details
{
 
template<class A1> struct storage1 {
	explicit storage1(A1 a1) :
			a1_(a1) {
		cout<<"storage1  storage1(A1 a1)"<<endl;
	}
	A1 a1_;
};
template<int I> struct storage1<boost::arg<I> > {
	explicit storage1(boost::arg<I>) {
		cout<<"storage1  storage1(boost::arg<I>)"<<endl;
	}
	static boost::arg<I> a1_() {
		return boost::arg<I>();
	}
};
template<int I> struct storage1<boost::arg<I> (*)()> {
	explicit storage1(boost::arg<I> (*)()) {
		cout<<"storage1  storage1(boost::arg<I> (*)())"<<endl;
	}
	static boost::arg<I> a1_() {
		return boost::arg<I>();
	}
};
// 2
template<class A1, class A2> struct storage2: public storage1<A1> {
	typedef storage1<A1> inherited;
	storage2(A1 a1, A2 a2) :
			storage1<A1>(a1), a2_(a2) {
		cout<<"storage2  storage2(A1 a1, A2 a2)"<<endl;
	}
	A2 a2_;
};
template<class A1, int I> struct storage2<A1, boost::arg<I> > : public storage1<
		A1> {
	typedef storage1<A1> inherited;
	storage2(A1 a1, boost::arg<I>) :
			storage1<A1>(a1) {
		cout<<"storage2  storage2(A1 a1, boost::arg<I>)"<<endl;
	}
	static boost::arg<I> a2_() {
		return boost::arg<I>();
	}
};
template<class A1, int I> struct storage2<A1, boost::arg<I> (*)()> : public storage1<
		A1> {
	typedef storage1<A1> inherited;
	storage2(A1 a1, boost::arg<I> (*)()) :
			storage1<A1>(a1) {
		cout<<"storage2  storage2(A1 a1, boost::arg<I> (*)())"<<endl;
	}
	static boost::arg<I> a2_() {
		return boost::arg<I>();
	}
};
// result_traits
template<class R, class F> struct result_traits {
	typedef R type;
};
struct unspecified {
};
template<class F> struct result_traits<unspecified, F> {
	typedef typename F::result_type type;
};
// value
template<class T> class value {
public:
	value(T const & t) :
			t_(t) {
	}
	T & get() {
		return t_;
	}
private:
	T t_;
};
// type
template<class T> class type {
};
// unwrap
template<class F> struct unwrapper {
	static inline F & unwrap(F & f, long) {
		return f;
	}
};
// listN
class list0 {
public:
	list0() {
	}
	template<class T> T & operator[](_bi::value<T> & v) const {
		cout << "list0  T & operator[](_bi::value<T> & v)" << endl;
		return v.get();
	}
	template<class R, class F, class A> R operator()(type<R>, F & f, A &,
			long) {
		cout << "list0  R operator()(type<R>, F & f, A &, long)" << endl;
		return unwrapper<F>::unwrap(f, 0)();
	}
	template<class F, class A> void operator()(type<void>, F & f, A &, int) {
		cout << "list0  void operator()(type<void>, F & f, A &, int)" << endl;
		unwrapper<F>::unwrap(f, 0)();
	}
};
 
template<class A1> class list1: private storage1<A1> {
private:
	typedef storage1<A1> base_type;
public:
	explicit list1(A1 a1) :
			base_type(a1) {
	}
	A1 operator[](boost::arg<1>) const {
		return base_type::a1_;
	}
	A1 operator[](boost::arg<1> (*)()) const {
		return base_type::a1_;
	}
	template<class T> T & operator[](_bi::value<T> & v) const {
		return v.get();
	}
	template<class R, class F, class A> R operator()(type<R>, F & f, A & a,
			long) {
		return unwrapper<F>::unwrap(f, 0)(a[base_type::a1_]);
	}
//	template<class F, class A> void operator()(type<void>, F & f, A & a, int) {
//		unwrapper<F>::unwrap(f, 0)(a[base_type::a1_]);
//	}
};
 
template<class A1, class A2> class list2: private storage2<A1, A2> {
private:
	typedef storage2<A1, A2> base_type;
public:
	list2(A1 a1, A2 a2) :
			base_type(a1, a2) {
	}
	A1 operator[](boost::arg<1>) const {
		cout << "list2  A1 operator[](boost::arg<1>)" << endl;
		return base_type::a1_;
	}
	A2 operator[](boost::arg<2>) const {
		cout << "list2  A1 operator[](boost::arg<2>)" << endl;
		return base_type::a2_;
	}
	A1 operator[](boost::arg<1> (*)()) const {
		cout << "list2  A1 operator[](boost::arg<1> (*)())" << endl;
		return base_type::a1_;
	}
	A2 operator[](boost::arg<2> (*)()) const {
		cout << "list2  A1 operator[](boost::arg<2> (*)())" << endl;
		return base_type::a2_;
	}
	template<class T> T & operator[](_bi::value<T> & v) const {
		cout << "T & operator[](_bi::value<T> & v)" << endl;
		return v.get();
	}
	template<class R, class F, class A> R operator()(type<R>, F & f, A & a,
			long) {
		return f(a[base_type::a1_], a[base_type::a2_]);
//		return unwrapper<F>::unwrap(f, 0)(a[base_type::a1_], a[base_type::a2_]);
	}
//	template<class F, class A> void operator()(type<void>, F & f, A & a, int) {
//		unwrapper<F>::unwrap(f, 0)(a[base_type::a1_], a[base_type::a2_]);
//	}
};
// bind_t
template<class R, class F, class L> class bind_t {
public:
	typedef bind_t this_type;
	bind_t(F f, L const & l) :
			f_(f), l_(l) {
	}
	typedef typename result_traits<R, F>::type result_type;
	result_type operator()() {
		cout << "bind_t::result_type operator()()" << endl;
		list0 a;
		return l_(type<result_type>(), f_, a, 0);
	}
	template<class A1> result_type operator()(A1 & a1) {
		list1<A1 &> a(a1);
		return l_(type<result_type>(), f_, a, 0);
	}
	template<class A1, class A2> result_type operator()(A1 & a1, A2 & a2) {
		list2<A1 &, A2 &> a(a1, a2);
		return l_(type<result_type>(), f_, a, 0);
	}
private:
	F f_;
	L l_;
};
 
template<class T, int I> struct add_value_2 {
	typedef boost::arg<I> type;
};
 
template<class T> struct add_value_2<T, 0> {
	typedef _bi::value<T> type;
};
template<class T> struct add_value {
	typedef typename add_value_2<T, boost::is_placeholder<T>::value>::type type;
};
template<class T> struct add_value<value<T> > {
	typedef _bi::value<T> type;
};
template<int I> struct add_value<arg<I> > {
	typedef boost::arg<I> type;
};
//template<int I> struct add_value<arg<I> (*)()> {
//	typedef boost::arg<I> (*type)();
//};
template<class R, class F, class L> struct add_value<bind_t<R, F, L> > {
	typedef bind_t<R, F, L> type;
};
// list_av_N
template<class A1> struct list_av_1 {
	typedef typename add_value<A1>::type B1;
	typedef list1<B1> type;
};
template<class A1, class A2> struct list_av_2 {
	typedef typename add_value<A1>::type B1;
	typedef typename add_value<A2>::type B2;
	typedef list2<B1, B2> type;
};
} // namespace _bi
 
// function pointers
template<class R>
_bi::bind_t<R, R (*)(), _bi::list0> bind2(R (*f)()) {
	typedef R (*F)();
	typedef _bi::list0 list_type;
	return _bi::bind_t<R, F, list_type>(f, list_type());
}
template<class R, class B1, class A1>
_bi::bind_t<R, R (*)(B1), typename _bi::list_av_1<A1>::type> bind2(R (*f)(B1),
		A1 a1) {
	typedef R (*F)(B1);
	typedef typename _bi::list_av_1<A1>::type list_type;
	return _bi::bind_t<R, F, list_type>(f, list_type(a1));
}
template<class R, class B1, class B2, class A1, class A2>
_bi::bind_t<R, R (*)(B1, B2), typename _bi::list_av_2<A1, A2>::type> bind2(
		R (*f)(B1, B2), A1 a1, A2 a2) {
	typedef R (*F)(B1, B2);
	typedef typename _bi::list_av_2<A1, A2>::type list_type;
	return _bi::bind_t<R, F, list_type>(f, list_type(a1, a2));
}
} // namespace boost
 
namespace {
boost::arg<1> _1;
boost::arg<2> _2;
}
#endif // #ifndef BOOST_BIND_BIND_HPP_INCLUDED__Mybind
```

main.cpp：

```cpp
#include <iostream>
#include "myBind.h"
using namespace std;
void tow_arguments(int i1, int i2) {
	std::cout << i1 << i2 << '\n';
}
 
class Test{
};
void testClass(Test t, int i){
	cout<<"testClass,Test,int"<<endl;
}
 
int main() {
	int i1 = 1, i2 = 2;
 
	(boost::bind2(&tow_arguments, 123, _1))(i1, i2);
	(boost::bind2(&tow_arguments, _1, _2))(i1, i2);
	(boost::bind2(&tow_arguments, _2, _1))(i1, i2);
	(boost::bind2(&tow_arguments, _1, _1))(i1, i2);
 
	(boost::bind2(&tow_arguments, 222, 666))(i1, i2);
 
	Test t;
	(boost::bind2(&testClass, _1, 666))(t, i2);
	(boost::bind2(&testClass, _1, 666))(t);
	(boost::bind2(&testClass, t, 666))();
}
```

在上面的代码中有很多个类，有几个是比较重要的any，storage，list和bind_t。

先来看类any：

```cpp
template<int I> struct arg {
	arg() {
	}
};
namespace {
boost::arg<1> _1;
boost::arg<2> _2;
}
```

在上面的代码中，我们看到了_1和_2，没错，所谓的占位符就是这么个东东，在匿名名字空间中定义的空的struct。
接下来是storage1,storage2类：

```cpp
template<class A1> struct storage1 {
	explicit storage1(A1 a1) :
			a1_(a1) {
	}
 
	A1 a1_;
};
...
// 2
template<class A1, class A2> struct storage2: public storage1<A1> {
	typedef storage1<A1> inherited;
 
	storage2(A1 a1, A2 a2) :
			storage1<A1>(a1), a2_(a2) {
	}
 
	A2 a2_;
};
...
```

可以看出storage类只是简单的继承关系，**实际上storage类是用来存储bind函数所传递进来的参数的值的**，之所以采用继承而不用组合，其实有精妙的用处。

接着看类list1和list2，**它俩分别是storage1和storage2的子类（即参数是存放在listN类中的）：**

```cpp
template<class A1> class list1: private storage1<A1> {
...
}
template<class A1, class A2> class list2: private storage2<A1, A2> {
...
}
```

仔细看代码，可以看到 **list1和list2重载了operator [ ] 和operator () 函数**，这些重载函数很关键，下面会谈到。

再看类bind_t：

```cpp
template<class R, class F, class L> class bind_t {
public:
	typedef bind_t this_type;
	bind_t(F f, L const & l) :
			f_(f), l_(l) {
	}
	typedef typename result_traits<R, F>::type result_type;
 
	result_type operator()() {
		cout<<"bind_t::result_type operator()()"<<endl;
		list0 a;
		return l_(type<result_type>(), f_, a, 0);
	}
...
private:
	F f_;   //bind所绑定的函数
	L l_;   //实际是是listN类，存放bind传绑定的参数
};
```

同样，我们可以看到bind_t类重载了operator ()函数，实际上，bind_t类是bind函数的返回类型，bind_t实际上是一个stl意义上的funtor。
注意bind_t的两个成员F f_ 和 L l_，这里是关键之处。



介绍完关键类，下面以main.cpp中下面部分代码为例，详细说明它的工作流程。

```cpp
	int i1 = 1, i2 = 2;
	(boost::bind2(&tow_arguments, 123, _1))(i1, i2);
```

首先bind2函数返回一个bind_t类，这个类中的F成员，保存了tow_arguments函数指针，L成员（即list2类），保存了参数123 和 _1。
bind_t类采用的是以下的特化：

```cpp
template<class R, class B1, class B2, class A1, class A2>
_bi::bind_t<R, R (*)(B1, B2), typename _bi::list_av_2<A1, A2>::type> bind2(
		R (*f)(B1, B2), A1 a1, A2 a2)
```

其中storage2类（list2的父类）和storage1类，分别采用的是以下的特化：

```cpp
template<class A1, int I> struct storage2<A1, boost::arg<I> > : public storage1<A1>
```

```cpp
template<class A1> struct storage1
```

（这里可以试下把123和_1互换位置，就能发现storage系列类用继承的精妙之处了，这里不展开了）

当bind_t调用operator (i1, i2)函数时，即以下函数：

```cpp
	template<class A1, class A2> result_type operator()(A1 & a1, A2 & a2) {
		list2<A1 &, A2 &> a(a1, a2);
		return l_(type<result_type>(), f_, a, 0); //operator ()
	}
```

再次生成一个list2，这个list2中保存的是i1 和 i2的值。接着调用`l_.operator() (type<result_type>(), f_, a, 0)`函数（还记得l_是什么东东不？）

**l_实际是上刚才保存了123和_1的list2！而f_是由bind函数据绑定的函数的指针！**

接着看list2的operator() 函数的实现：

```cpp
	template<class R, class F, class A> R operator()(type<R>, F & f, A & a,
			long) {
		return f(a[base_type::a1_], a[base_type::a2_]);
//		return unwrapper<F>::unwrap(f, 0)(a[base_type::a1_], a[base_type::a2_]); //本是这句的，简化了下，无关要紧
	}
```

可以看到f是bind所绑定的函数的指针，即这里是调用了我们所绑定的函数。那么参数是从哪里来的？
注意A &a实际上是保存了i1和i2的list2！，所以这里又调用了list2的operator[ ]函数！

再看下list2的operator[] 函数的实现：

```cpp
	A1 operator[](boost::arg<1>) const {
		cout << "list2  A1 operator[](boost::arg<1>)" << endl;
		return base_type::a1_;
	}
	A2 operator[](boost::arg<2>) const {
		cout << "list2  A1 operator[](boost::arg<2>)" << endl;
		return base_type::a2_;
	}
 
	A1 operator[](boost::arg<1> (*)()) const {
		cout << "list2  A1 operator[](boost::arg<1> (*)())" << endl;
		return base_type::a1_;
	}
	A2 operator[](boost::arg<2> (*)()) const {
		cout << "list2  A1 operator[](boost::arg<2> (*)())" << endl;
		return base_type::a2_;
	}
 
	template<class T> T & operator[](_bi::value<T> & v) const {
		cout << "T & operator[](_bi::value<T> & v)" << endl;
		return v.get();
	}
```

貌似问题都解决了，其实这里还有一个关键要素，两个list2是怎么合并起来，组成正确的参数传递给函数f的？（第一个list2存放了123和_1，第二个list2存放了i1和i2）
传递给tow_arguments函数的参数实际是是（123, 1）！

这里要仔细研究这些operator[]函数的实现，才能真正理解其过程。


**注意，上面的的代码中bind2只能绑定普通的函数，不能绑定类成员函数，functor，智能指针等。不过其实区别不大，只是增加一些特化的代码而已。**

还有一些const相关的函数删掉了。

## 后记

从代码可以看到boost::bind和原生的函数指针相比，会损失效率（两次的寻址，函数参数的拷贝，有一些可能没有inline的函数调用）。

不得不吐槽下boost的代码中那些神奇的宏，如果不是IDE有提示，我想真心弄不明白到底哪段是有意义的。。有时候还很神奇地include一部分代码进来（不是头文件，只是实现的一部分！）。

不得不吐槽下模板编程中的const，为了一个函数，比如int sum(int a, int b); 就得写四个重载函数来对应不同参数是或者不是const的情况。所以大家可以想像bind最多支持9个参数，那么有多少种情况了。

boost::bind的实现的确非常精巧，隐藏了很多复杂性。在《C++沉思录》中作者说到世界是复杂的，可以通过复杂性获取简单性。也许这个是对的，但是对于无数的后来的程序员总会有人想要看看黑盒子里到底是什么东东，结果总要花大量的时间才能理解，才能获得这种“简单性”。

C++的模板代码中最痛苦的是到处都是typedef，一个typedef就把所有的类型信息都干掉了！而这东东又是必须的。

也许C++给编程语言技术带来的最大贡献就是牛B的编译器了！

C++11中貌似部分的支持concept，希望这个能简化模板编程。

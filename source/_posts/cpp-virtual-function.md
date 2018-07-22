---
title: C++中虚函数(virtual function)到底有多慢
date: 2012-04-16 21:58:28
tags:
 - cpp


categories:
  - 技术

---


## 前言

虚函数为什么慢，cpu分支预测技术，虚函数到底要调用哪些汇编，虚函数实现的简单图示，虚函数不能内联，

印象中经常看到有人批评C++的虚函数很慢，但是虚函数为什么慢，虚函数到底有多慢呢？

## 理论分析

虚函数慢的原因主要有三个：

* 多了几条汇编指令（运行时得到对应类的函数的地址）
* 影响cpu流水线
* 编译器不能内联优化（仅在用父类引用或者指针调用时，不能内联）

先简单说下虚函数的实现，以下面测试代码中的VirtualVector类为例，VirtualVector类的内存布局如下：

![](/img/cpp-vtable.jpg)

当在用父类的引用或者指针调用虚函数时，会先从该对象的头部取到虚函数的地址（C++标准规定虚函数表地址必须放最前），再从虚函数表中取到实际要调用的函数的地址，最终调用该函数，汇编代码大概如下：

```cpp
			sum += v.at(i);       //要调用at函数
00CF1305  mov         eax,dword ptr [ebx]     //取到对象的虚函数表地址
00CF1307  mov         edx,dword ptr [eax+4]   //取到实际VirtualVector类的at函数地址，因为at是第二个虚函数，所以要+4，如果是clear则+8，push_back则不加
00CF130A  push        esi                     //参数压栈
00CF130B  mov         ecx,ebx                 
00CF130D  call        edx                     //调用真正的VirtualVector类的at函数
```

**所以，我们可以看到调用虚函数，相比普通函数，实际上多了三条汇编指令（取虚表，取函数地址，call调用）。**

**至于虚函数如何影响cpu的流水线，则主要是因为call指令**，具体可以看这个网址的演示：

CPU流水线的一个演示：http://www.pictutorials.com/Instruction_Pipeline.html

第3点，编译器不能内联优化，则是因为在得到子类的引用或者指针之前，根本不知道要调用哪一个函数，所以无从内联。

但是，要注意的是，对于子类直接调用虚函数，是可以内联优化的。如以下的代码，编译器是完全可以内联展开的。

```cpp
	VirtualVector v(100);
	v.push_back(1);
	v.at(0);
```

## 实际测试

光说不练显然不行，下面用代码来测试下虚函数到底有多慢。

下面的代码只是测试用，不考虑细节。

Vector类包装了一个数组，提供push_back，at，clear函数。

VirtualVector类继承了IVector，同样实现了push_back，at，clear函数，但是都是虚函数。

```cpp
#include <iostream>
#include <time.h>
#include <vector>
using namespace std;
 
const int size = 100000000;
 
class Vector{
private:
	int *array;
	int pos;
public:
	Vector(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
};
class IVector{
public:
	virtual void push_back(int val) = 0;
	virtual int at(int i) = 0;
	virtual void clear() = 0;
	virtual ~IVector() {};
};
 
class VirtualVector : public IVector{
public:
	int *array;
	int pos;
public:
	VirtualVector(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
	~VirtualVector()
	{
		if(array != NULL)
			delete array;
	}
};
 
void testVectorPush(Vector& v){
	v.clear();
 
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVectorAt(Vector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
void testVirtualVectorPush(IVector& v)
{
	v.clear();
 
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVirtualVectorAt(IVector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
int main()
{
	cout<<sizeof(VirtualVector)<<endl;
 
 
	Vector *v = new Vector(size);
	VirtualVector *V = new VirtualVector(size);
 
	cout<<"testVectorPush:"<<endl;
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
 
	cout<<"testVirtualVectorPush:"<<endl;
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
 
	cout<<"testVectorAt:"<<endl;
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
 
	cout<<"testVirtualVectorAt:"<<endl;
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
 
	return 0;
}
```

上面的是只有一层继承的情况时的结果，**尽管从虚函数的实现角度来看，多层继承和一层继承调用虚函数的效率都是一样的。**

但是为了测试结果更加可信，下面是一个6层继承的测试代码（为了防止编译器的优化，有很多垃圾代码）：


```cpp
#include <iostream>
#include <time.h>
#include <vector>
using namespace std;
 
const int size = 100000000;
 
class Vector{
private:
	int *array;
	int pos;
public:
	Vector(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
};
class IVector{
public:
	virtual void push_back(int val) = 0;
	virtual int at(int i) = 0;
	virtual void clear() = 0;
	virtual ~IVector() {};
};
 
class VirtualVector1 : public IVector{
public:
	int *array;
	int pos;
public:
	VirtualVector1(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[1] = val;
	}
	int at(int i)
	{
		return array[1];
	}
	void clear()
	{
		pos = 0;
	}
	~VirtualVector1()
	{
		if(array != NULL)
			delete array;
	}
};
 
class VirtualVector2 : public VirtualVector1{
public:
	VirtualVector2(int size):VirtualVector1(size)
	{
	}
	void push_back(int val)
	{
		array[2] = val;
	}
	int at(int i)
	{
		return array[2];
	}
	void clear()
	{
		pos = 0;
	}
};
 
class VirtualVector3 : public VirtualVector2{
public:
	VirtualVector3(int size):VirtualVector2(size)
	{
	}
	void push_back(int val)
	{
		array[3] = val;
	}
	int at(int i)
	{
		return array[3];
	}
	void clear()
	{
		pos = 0;
	}
};
 
class VirtualVector4 : public VirtualVector3{
public:
	VirtualVector4(int size):VirtualVector3(size)
	{
	}
	void push_back(int val)
	{
		array[4] = val;
	}
	int at(int i)
	{
		return array[4];
	}
	void clear()
	{
		pos = 0;
	}
};
 
class VirtualVector5 : public VirtualVector4{
public:
	VirtualVector5(int size):VirtualVector4(size)
	{
	}
	void push_back(int val)
	{
		array[5] = val;
	}
	int at(int i)
	{
		return array[5];
	}
	void clear()
	{
		pos = 0;
	}
};
 
class VirtualVector6 : public VirtualVector5{
public:
	VirtualVector6(int size):VirtualVector5(size)
	{
	}
	void push_back(int val)
	{
		array[6] = val;
	}
	int at(int i)
	{
		return array[6];
	}
	void clear()
	{
		pos = 0;
	}
};
 
class VirtualVector : public VirtualVector6{
public:
	VirtualVector(int size):VirtualVector6(size)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
};
 
void testVectorPush(Vector& v){
	v.clear();
 
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVectorAt(Vector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
void testVirtualVectorPush(IVector& v)
{
	v.clear();
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVirtualVectorAt(IVector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
int main()
{
	cout<<sizeof(VirtualVector)<<endl;
	{
		auto v = VirtualVector1(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
	{
		auto v = VirtualVector2(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
	{
		auto v = VirtualVector3(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
	{
		auto v = VirtualVector4(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
	{
		auto v = VirtualVector5(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
	{
		auto v = VirtualVector6(size);
		v.push_back(0);
		cout<<v.at(0)<<endl;
	}
 
	auto *v = new Vector(size);
	auto *V = new VirtualVector(size);
 
	cout<<"testVectorPush:"<<endl;
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
 
	cout<<"testVirtualVectorPush:"<<endl;
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
 
	cout<<"testVectorAt:"<<endl;
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
 
	cout<<"testVirtualVectorAt:"<<endl;
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
 
	return 0;
}
```

## 测试结果

测试结果都取最小时间

1层继承的测试结果：

| |push_back | at|
|---|---|---|
|Vector|0.263s|0.04s|
|VirtualVector|0.331s|0.222s|
|倍数|1.25|5.55|

6层继承的测试结果：

| |push_back | at|
|---|---|---|
|Vector|0.262s|0.041s|
|VirtualVector|0.334s|0.223s|
|倍数|1.27|5.43|

1. 可以看出继承层数和虚函数调用效率无关

2. 可以看出虚函数慢得有点令人发指了，对于at操作，虚函数花的时间竟然是普通函数的5.5倍！

但是，再看看，我们可以发现对于push_back操作，虚函数花的时间是普通函数的1.25倍。why?

再分析下代码，我们可以发现at操作的逻辑，明显要比push_back的逻辑要简单。

虚函数额外消耗时间为 vt，函数本身所消耗时间为 ft，则有以下

* 倍数 = (vt + ft)/ft = 1 + vt/ft

显然当ft越大，即函数本身消耗时间越长，则倍数越小。

那么让我们在at操作中加了额外代码，统计下1到100之和：

```cpp
	int at(int i)
	{
		sssForTest = 0;
		for(int j = 0; j < 100; ++j)
			sssForTest += j;
		return array[i];
	}
```

测试代码：

```cpp
#include <iostream>
#include <time.h>
#include <vector>
using namespace std;
 
const int size = 100000000;
int sssForTest = 0;
 
class Vector{
private:
	int *array;
	int pos;
public:
	Vector(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		sssForTest = 0;
		for(int j = 0; j < 100; ++j)
			sssForTest += j;
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
};
class IVector{
public:
	virtual void push_back(int val) = 0;
	virtual int at(int i) = 0;
	virtual void clear() = 0;
	virtual ~IVector() {};
};
 
class VirtualVector : public IVector{
public:
	int *array;
	int pos;
public:
	VirtualVector(int size):array(new int[size]),pos(0)
	{
	}
	void push_back(int val)
	{
		array[pos++] = val;
	}
	int at(int i)
	{
		sssForTest = 0;
		for(int j = 0; j < 100; ++j)
			sssForTest += j;
		return array[i];
	}
	void clear()
	{
		pos = 0;
	}
	~VirtualVector()
	{
		if(array != NULL)
			delete array;
	}
};
 
void testVectorPush(Vector& v){
	v.clear();
 
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVectorAt(Vector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
void testVirtualVectorPush(IVector& v)
{
	v.clear();
 
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	nTimeStart = clock();    //
	for(int i = 0; i < size; ++i)
	{
		v.push_back(i);
		//cout<<v.size()<<endl;
	}
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
}
 
void testVirtualVectorAt(IVector& v)
{
	clock_t nTimeStart;      //计时开始
	clock_t nTimeStop;       //计时结束
	int sum = 0;
	nTimeStart = clock();    //
	for(int j = 0; j < 1; ++j)
	{
		for(int i = 0; i < size; ++i)
		{
			sum += v.at(i);
		}
	}
 
	nTimeStop = clock();    //
	cout <<"耗时："<<(double)(nTimeStop - nTimeStart)/CLOCKS_PER_SEC<<"秒"<< endl;
	cout<<"sum:"<<sum<<endl;
}
 
int main()
{
	cout<<sizeof(VirtualVector)<<endl;
 
 
	Vector *v = new Vector(size);
	VirtualVector *V = new VirtualVector(size);
 
	cout<<"testVectorPush:"<<endl;
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
	testVectorPush(*v);
 
	cout<<"testVirtualVectorPush:"<<endl;
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
	testVirtualVectorPush(*V);
 
	cout<<"testVectorAt:"<<endl;
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
	testVectorAt(*v);
 
	cout<<"testVirtualVectorAt:"<<endl;
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
	testVirtualVectorAt(*V);
 
	return 0;
}
```

at操作中增加求和后的统计结果：

| |push_back | at 增加求和代码|
|---|---|---|
|Vector|0.265s|6.893s|
|VirtualVector|0.328s|7.125s|
|倍数|1.23|1.03|

只是简单增加了一个求和代码，我们可以看到，倍数变成了1.03，也就是说虚函数的消耗基本可以忽略了。

**所以说，虚函数的效率到底低不低和实际要调用的函数的耗时有关，当函数本身的的耗时越长，则虚函数的影响则越小。**

再从另一个角度来看，一个虚函数调用到底额外消耗了多长时间？

从统计数据来看100,000,000次函数调用，虚函数总共额外消耗了0.05~0.23秒（VirtualVector对应时间减去Vector时间），

**也就是说，1亿次调用，虚函数额外花的时间是0.x到2.3秒。**

也就是说，如果你有个函数，要被调用1亿次，而这1亿次调用所花的时间是几秒，十几秒，且你不能容忍它慢一二秒，那么就干掉虚函数吧^_^。

## 总结

* 虚函数调用效率和继承层数无关；
* 其实虚函数还是挺快的。
* 如果真的要完全移除虚函数，那么如果要实现运行时多态，则要用到函数指针，据上面的分析，函数指针基本具有虚函数的所有缼点（要传递函数指针，同样无法内联，同样影响流水线），且函数指针会使代码混乱。

BTW：测试cpu是i5。

TODO：测试指针函数，boost::bind的效率
---
layout:     post
title:      C++ 对象模型
subtitle:   <<深度探索C++对象模型>> 读后总结
date:       2020-03-21
author:     jiuguang
catalog: true
tags:
    - C++ 思考
---
## 前言

重温了一遍<b><<深度探索C++对象模型>></b>，多了一些自己的思考，本文做一些记录

### 什么是"类"？

站在纯编码的角度上看，类就是组织数据结构的一种方式，那么下列都可以称为"类":
+ C语言种的struct，甚至uion
+ Erlang 中的复合结构：Term=>Tuple,list,map等
+ Lua 中的table
+ C#/Java 中的Class
+ C++ 中的class
+ .....

站在编程范式的角度上，类就是面向对象范式中的"对象描述"，或者说等同于"class"。那么class与单纯的"数据组织"（用struct泛指）方式区别又在哪里：
+ class具有"行为"，struct不具备
+ class具有访问属性，对应面向对象中：封装
+ class可以被继承，对应面向对象中：继承/多态

#### C++简单Class模型
那么为了class这些特性，C++构造一个class付出了哪些成本？----答案是：几乎没有。对于这个简单class，它在内存中的布局具体具体是怎样的呢?
```
#include <iostream>

class Point
{
private:
	float x;
	float y;
public:
	Point(float _x, float _y): 
		x(_x), y(_y) { }
	float getX() { return x; }
	float getY() { return y; }
};

int main(int argc, char const *argv[])
{
	std::cout<<"Point Size: "<<sizeof(Point)<<std::endl;
	return 0;
}

/*
output:
Point Size: 8
*/
```

在64位sys，gcc编译后输出8，说明Point只付出了data成员的成本。那么在Point中 data和function分别怎么处理的呢
+ data数据会像struct一样，真正被存于对象中，占对应的内存size
+ function会作为全局function处理，不跟具体object有任何关系
![](https://raw.githubusercontent.com/lixiang-share/lixiang-share.github.io/master/img/cpp_class_model_point.png)

那么为什么我们能在成员function中对成员data做直接访问呢？
+ object在调用成员func时会默认把自身作为参数传进去，所以上述```getX/getY```的签名会被编译器重新处理

```
float getX(Point* this) { return this->x; }
float getY(Point* this) { return this->y; }
```

+ 成员func相互调用也是基于此原理

#### 多态下class模型

C++发生继承后，object model 会有怎样的变化呢？

```
//--- 为了不牵涉到对齐问题，64位system上，把float改为double，保持和指针大小一致
#include <iostream>

class Point
{
public:
	double x;
	double y;
public:
	Point(double _x, double _y): 
		x(_x), y(_y) { }
	double getX() { return x; }
	double getY() { return y; }
	virtual double getSqurtLength() { return x*x + y*y; }
};

class Point3d: public Point
{
public:
	double z;
public:
	Point3d(double _x, double _y, double _z): 
		Point(_x, _y), z(_z) { }
	virtual double getSqurtLength() { return x*x + y*y * z*z; }
};

int main(int argc, char const *argv[])
{
	Point3d p3d(1.0, 2.0, 3.0);
	std::cout<<"Point3d Size: "<<sizeof(p3d)<<std::endl;
	return 0;
}

/*
output:
Point Size: 32
*/
```

按照上述，一共三个double类型成员 3*8 = 24才对，但却多出8字节，这个成本就是c++实现多态的核心机制：<b>vtbl</b>，Point3d内存布局如下图：

![](https://raw.githubusercontent.com/lixiang-share/lixiang-share.github.io/master/img/cpp_class_model_virtual.png)

+ 继承过程中如果没有virtual function，那么付出的成本仅仅只有成员内存
+ typeinfo 是C++运行期对class的描述信息 [type_info](https://zh.cppreference.com/w/cpp/types/type_info)


### 什么是"编译器需要"？

### "继承"做了哪些事？

### "成员访问"做了哪些事？

### "运行期"做了哪些事？


## 总结
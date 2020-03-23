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

"C++ class如果没有定义Constructor/Destructor，那么编译器就会自动生成一个"，这个理论一定正确吗？----> 答：并不是，准确说仅仅当"编译器需要"时才会自动生成。

```
class Empty
{
private:
	float x;
public:
	Empty() { x = 1;};
	~Empty() {};
};

int main(int argc, char const *argv[])
{
	Empty e;
	return 0;
}
```
用```g++ -c Empty.cpp -o Empty.o```编译出目标对象，然后```readelf -s Empty.o```查看symtbl，直观就可以看到，constructor/destructor 被mangling后的符号
```
Symbol table '.symtab' contains 22 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Empty.cpp
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    9 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT   10 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT   12 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT   13 
    10: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    1 _ZN5EmptyC5Ev
    11: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    2 _ZN5EmptyD5Ev
    12: 0000000000000000     0 SECTION LOCAL  DEFAULT   11 
    13: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
    14: 0000000000000000     0 SECTION LOCAL  DEFAULT    2 
    15: 0000000000000000    27 FUNC    WEAK   DEFAULT    7 _ZN5EmptyC2Ev
    16: 0000000000000000    27 FUNC    WEAK   DEFAULT    7 _ZN5EmptyC1Ev
    17: 0000000000000000    11 FUNC    WEAK   DEFAULT    9 _ZN5EmptyD2Ev
    18: 0000000000000000    11 FUNC    WEAK   DEFAULT    9 _ZN5EmptyD1Ev
    19: 0000000000000000    89 FUNC    GLOBAL DEFAULT    3 main
    20: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    21: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND __stack_chk_fail

```
然后如果删除代码中的constructor/destructor定义：
```
class Empty
{
private:
	float x;
};

int main(int argc, char const *argv[])
{
	Empty e;
	return 0;
}
```
重复上述过程，会发现constructor/destructor相关符号消失了：

```
Symbol table '.symtab' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Empty.cpp
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     8: 0000000000000000    18 FUNC    GLOBAL DEFAULT    1 main
```

那么编译器什么情况会生成默认constructor/destructor，
+ 成员变量 具有default constructor --> 构造时必须调成员的构造方法
+ Base class 具有default constructor --> 构造时必须调base 构造方法
+ 带有virtual 方法的class --> 构造时必须初始化vtbl
+ base class 具有virtual 方法 --> 构造时必须初始化vtbl

简单说，只有”编译器需要“的时候才会调整代码结构，插入它需要的代码段。上述结论用virtual base 做个案例：
```
class base
{
public:
	virtual void print(){}
};

class deriver: public base
{
public:
	int a;
	void print() { a = 2;}
};

int main(int argc, char const *argv[])
{
	deriver d;
	return 0;
}
```
用readefl查看符号表
```
Symbol table '.symtab' contains 47 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Empty.cpp
	....
    17: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    3 _ZN4baseC5Ev
    18: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    4 _ZN7deriverC5Ev
 	.....
    29: 0000000000000000     0 SECTION LOCAL  DEFAULT   10 
    30: 0000000000000000    11 FUNC    WEAK   DEFAULT   15 _ZN4base5printEv
    31: 0000000000000000    22 FUNC    WEAK   DEFAULT   16 _ZN7deriver5printEv
    32: 0000000000000000    25 FUNC    WEAK   DEFAULT   17 _ZN4baseC2Ev
    33: 0000000000000000    24 OBJECT  WEAK   DEFAULT   23 _ZTV4base
    34: 0000000000000000    25 FUNC    WEAK   DEFAULT   17 _ZN4baseC1Ev
    35: 0000000000000000    41 FUNC    WEAK   DEFAULT   19 _ZN7deriverC2Ev
    36: 0000000000000000    24 OBJECT  WEAK   DEFAULT   21 _ZTV7deriver
    37: 0000000000000000    41 FUNC    WEAK   DEFAULT   19 _ZN7deriverC1Ev
    38: 0000000000000000    69 FUNC    GLOBAL DEFAULT   11 main
    39: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    40: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND __stack_chk_fail
    41: 0000000000000000    24 OBJECT  WEAK   DEFAULT   25 _ZTI7deriver
    42: 0000000000000000    16 OBJECT  WEAK   DEFAULT   28 _ZTI4base
    43: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZTVN10__cxxabiv120__si_c
    44: 0000000000000000     9 OBJECT  WEAK   DEFAULT   27 _ZTS7deriver
    45: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZTVN10__cxxabiv117__clas
    46: 0000000000000000     6 OBJECT  WEAK   DEFAULT   30 _ZTS4base
```




### "继承"做了哪些事？

### "成员访问"做了哪些事？

### "运行期"做了哪些事？


## 总结
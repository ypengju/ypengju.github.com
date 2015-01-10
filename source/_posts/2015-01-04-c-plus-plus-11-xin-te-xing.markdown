---
layout: post
title: "c++11 -- 新特性"
date: 2015-01-04 09:41:02 +0800
comments: true
categories: c++
---
在cocos2d-x 3.X之后的版本中，可以看到很多关于c++11新特性的使用，这篇文章学习记录下c++11的新特性及使用方法。    
###auto
####变量初始化
在c++11中，我们可以想其他动态语言一样，不需要指明一个变量的具体类型，直接对变量进行声明初始化。
```cpp  
auto i = 10;
auto str = "hello world";
auto p = Sprite::create("CloseNormal.png");
```   
这样在声明变量时，我们不需要指明变量的具体类型，但是在使用auto时，我们必须对变量赋值。同时我们还可以使用在STL的迭代器中，使得代码更加简单易读。<!--more-->
```cpp
std::map<std::string, std::vector<int>> map;
for(auto it = begin(map); it != end(map); ++it) 
{
}
```  
####函数返回值
auto不能用来声明函数的返回值。但如果函数有一个尾随的返回类型时，auto是可以出现在函数声明中返回值位置。
```cpp
template <typename T1, typename T2>
auto compose(T1 t1, T2 t2) -> decltype(t1 + t2)
{
   return t1+t2;
}
auto v = compose(2, 3.14); // v's type is double   
```   
此时，函数的返回值类型取决于两个参数相加后的数值类型。   
###nullptr
以前都是用0来表示空指针的，但由于0可以被隐式类型转换为整形，这就会存在一些问题。关键字nullptr是std::nullptr_t类型的值，用来指代空指针。nullptr和任何指针类型以及类成员指针类型的空值之间可以发生隐式类型转换，同样也可以隐式转换为bool型（取值为false）。但是不存在到整形的隐式类型转换。
```cpp
void func(int n); 
void func(char *s);
 
func( NULL ); // guess which function gets called?
```   
func传入的参数为NULL空指针，按理我们应该调用第二个函数，但是由于NULL就是0，而0是一个整形，所以实际调用的是第一个函数。使用nullptr就避免了这种问题，他明确指的是一个指针，而不会隐式转换为整形。  
###基于范围的for循环
之前使用for语句，我们必须指明下标，访问元素的个数。新的for循环语句像其他语言foreach语句一样，我们只关心遍历了数组中所有元素，而不用关心是按怎么样下标顺序进行访问。
```cpp
std::map<std::string, std::vector<int>> map;
std::vector<int> v;
v.push_back(1);
v.push_back(2);
v.push_back(3);
map["one"] = v;
 
for(const auto& kvp : map) 
{
  std::cout << kvp.first << std::endl;
 
  for(auto v : kvp.second)
  {
     std::cout << v << std::endl;
  }
}
 
int arr[] = {1,2,3,4,5};
for(int& e : arr) 
{
  e = e*e;
}
```
###override和final
在原来的情况因为疏忽或者不注意我们可能会造成一些微妙的错误。   
```cpp
class B 
{
public:
   virtual void f(short) {std::cout << "B::f" << std::endl;}
};
 
class D : public B
{
public:
   virtual void f(int) {std::cout << "D::f" << std::endl;}
};
```  
D::f 按理应当重写 B::f。然而二者的声明是不同的，一个参数是short，另一个是int。因此D::f（原文为B::f，可能是作者笔误——译者注）只是拥有同样名字的另一个函数（重载）而不是重写。当你通过B类型的指针调用f()可能会期望打印出D::f，但实际上则会打出 B::f 。   
另一个很微妙的错误情况：参数相同，但是基类的函数是const的，派生类的函数却不是。 
```cpp  
class B 
{
public:
   virtual void f(int) const {std::cout << "B::f " << std::endl;}
};
 
class D : public B
{
public:
   virtual void f(int) {std::cout << "D::f" << std::endl;}
};
```   
同样，这两个函数是重载而不是重写，所以你通过B类型指针调用f()将打印B::f，而不是D::f。

* override 表示函数应当重写基类中的虚函数。
* final 表示派生类不应当重写这个虚函数。   

使用上边两个新的标示符后，我们可以清晰的表达我们的意图。 
可以使用override明确表明此方法是否是父类方法
```cpp
class B 
{
public:
   virtual void f(int) {std::cout << "B::f " << std::endl;}
};
 
class D : public B
{
public:
   virtual void f(int) override {std::cout << "D::f" << std::endl;}    //OK
   virtual void f(double) override {std::cout << "D::f" << std::endl;} //Error
};
```
如果你希望函数不要再被派生类进一步重写，你可以把它标识为final。可以在基类或任何派生类中使用final。在派生类中，可以同时使用override和final标识。   
```cpp   
class B 
{
public:
   virtual void f(int) {std::cout << "B::f" << std::endl;}
};
 
class D : public B
{
public:
   virtual void f(int) override final {std::cout << "D::f" << std::endl;}
};
 
class F : public D
{
public:
   virtual void f(int) override {std::cout << "F::f" << std::endl;}
};
```   
被标记成final的函数将不能再被F::f重写。
###强类型枚举
传统的C++枚举类型存在一些缺陷：它们会将枚举常量暴露在外层作用域中（这可能导致名字冲突，如果同一个作用域中存在两个不同的枚举类型，但是具有相同的枚举常量就会冲突），而且它们会被隐式转换为整形，无法拥有特定的用户定义类型。

在C++11中通过引入了一个称为强类型枚举的新类型，修正了这种情况。强类型枚举由关键字enum class标识。它不会将枚举常量暴露到外层作用域中，也不会隐式转换为整形，并且拥有用户指定的特定类型（传统枚举也增加了这个性质）。   
```
enum class Options {None, One, All};
Options o = Options::All;
```
###Lambdas
Lambda表达式是一种描述函数对象的机制，它的主要应用是描述某些具有简单行为的函数（Lambda表达式也可以称为匿名函数)
####语法
1. [ capture-list ] ( params ) mutable(optional) exception attribute -> ret { body }    
2. [ capture-list ] ( params ) -> ret { body }
3. [ capture-list ] ( params ) { body }
4. [ capture-list ] { body }  

第一个为完整声明，根据需要参数，返回值等都可省略。
```cpp  
auto func1 = []{ std::cout << "hello world\n"; };  //省略参数的
auto func2 = [](int x, int y) { return x + y; } // 隐式返回类型
auto func3 = [](int x, int y) -> int { int z = x + y; return z; } //显示指定返回类型
```
Lambda函数可以引用在它之外声明的变量. 这些变量的集合叫做一个闭包. 闭包被定义在Lambda表达式声明中的方括号[]内. 这个机制允许这些变量被按值或按引用捕获。
```
[]        //未定义变量.试图在Lambda内使用任何外部变量都是错误的.
[x, &y]   //x 按值捕获, y 按引用捕获.
[&]       //用到的任何外部变量都隐式按引用捕获
[=]       //用到的任何外部变量都隐式按值捕获
[&, x]    //x显式地按值捕获. 其它变量按引用捕获
[=, &z]   //z按引用捕获. 其它变量按值捕获
```
下边是一些列子
```cpp
std::vector<int> v;
v.push_back(1);
v.push_back(2);
v.push_back(3);
 
std::for_each(std::begin(v), std::end(v), [](int n) {std::cout << n << std::endl;});
 
auto is_odd = [](int n) {return n%2==1;};
auto pos = std::find_if(std::begin(v), std::end(v), is_odd);
if(pos != std::end(v))
std::cout << *pos << std::endl;
```
在使用递归lambda时，我们不能使用auto来进行申明，因为auto对象类型由初始表达式决定，然而初始表达式又包含了对其自身的引用，因此要求先知道它的类型，这就导致了无穷递归。解决问题的关键就是打破这种循环依赖，用std::function显式的指定函数类型：
```cpp
auto fib = [&fib](int n) {return n < 2 ? 1 : fib(n-1) + fib(n-2);}; //Error

std::function<int(int)> lfib = [&lfib](int n) {return n < 2 ? 1 : lfib(n-1) + lfib(n-2);}; //OK
```

###参考
[C++开发者都应该使用的10个C++11特性](http://blog.jobbole.com/44015/)   
[C++11语言扩展：常规特性](http://blog.jobbole.com/55063/)   
[Lambda functions](http://en.cppreference.com/w/cpp/language/lambda)   
[C++11中的匿名函数(lambda函数,lambda表达式)](http://www.cnblogs.com/pzhfei/archive/2013/01/14/lambda_expression.html)



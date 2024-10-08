# 描述
***“替换失败不是错误” (Substitution Failure Is Not An Error)***
  
在函数模板的重载决议中会应用此规则：当模板形参在 替换 成显式指定的类型或推导出的类型失败时，从重载集中丢弃这个特化，而非导致编译失败。

此特性被用于模板元编程。

# 解释

对函数模板形参进行两次替换（由模板实参所替代）：

- 在模板实参推导前，对显式指定的模板实参进行替换
- 在模板实参推导后，对推导出的实参和从默认项获得的实参进行替换

替换发生于

- 函数类型中使用的所有类型（包括返回类型和所有形参的类型）
- 各个模板形参声明中使用的所有类型
- 部分特化的模板实参列表中使用的所有类型
- 函数类型中使用的所有表达式
  - 各个模板形参声明中使用的所有表达式
  - 部分特化的模板实参列表中使用的所有表达式==(C++11 起)==
- explicit 说明符中使用的所有表达式==(C++20 起)==

以上类型或表达式在以用来替换的实参写出时非良构（并带有必要的诊断）的任何场合，都是*替换失败*。

只有在函数类型或其模板形参类型或其 explicit 说明符==(C++20 起)==的*立即语境*中的类型与表达式中的失败，才是 SFINAE 错误。

如果对替换后的类型/表达式的求值导致副作用，例如实例化某模板特化、生成某隐式定义的成员函数等，那么这些副作用中的错误都被当做硬错误。

lambda 表达式不被当作是立即语境的一部分。==(C++20 起)==

替换按词法顺序进行，并在遇到失败时终止。

如果多个声明具有不同的词法顺序（例如，某个函数模板声明为具有尾随返回类型，它在某个形参之后替换，然后被重声明为具有常规返回类型，它则在该形参之前替换），而这会导致模板实例化以不同顺序出现或完全不出现，那么程序非良构；不要求诊断。(C++11 起)

```c++
template<typename A>
struct B { using type = typename A::type; };
 
template<class T,
    // 如果 T 没有成员 type 那么就是 SFINAE 失败
    class U = typename T::type,    
    // 如果 T 没有成员 type 那么就是硬错误
    class V = typename B<T>::type> 
    // （经由 CWG 1227 保证不出现，因为到 U 的默认模板实参中的替换会首先失败）
void foo (int);
 
template<class T>
typename T::type h(typename B<T>::type);
 
template<class T>
auto h(typename B<T>::type) -> typename T::type; // 重声明
 
template<class T>
void h(...) {}
 
using R = decltype(h<int>(0));// 非良构，不要求诊断
```

## 类型 SFINAE

下列类型的错误是 SFINAE 错误：

- 试图实例化含有多个不同长度的包的包展开(C++11 起)

- 试图创建 void 的数组，引用的数组，函数的数组，负大小的数组，非整型大小的数组，或者零大小的数组：
```c++
template<int I>
void div(char(*)[I % 2 == 0] = 0)
{
    // 当 I 是偶数时选择这个重载
}
 
template<int I>
void div(char(*)[I % 2 == 1] = 0)
{
    // 当 I 是奇数时选择这个重载
}
```

- 试图在作用域解析运算符 `**::**` 左侧使用类和枚举以外的类型：
```c++
template<class T>
int f(typename T::B*);
 
template<class T>
int f(T);
 
int i = f<int>(0); // 使用第二个重载
```
- 试图使用类型的成员，其中
  - 类型不包含指定成员
  - 在要求类型的地方，指定成员不是类型
  - 在要求模板的地方，指定成员不是模板
  - 在要求非类型的地方，指定成员不是非类型

```c++
template <int I>
struct X {};
 
template<template<class T> class>
struct Z {};
 
template<class T>
void f(typename T::Y*) {}
 
template<class T>
void g(X<T::N>*) {}
 
template<class T>
void h(Z<T::template TT>*) {}
 
struct A {};
struct B { int Y; };
struct C { typedef int N; };
struct D { typedef int TT; };
struct B1 { typedef int Y; };
struct C1 { static const int N = 0; };
struct D1
{ 
    template<typename T>
    struct TT {}; 
};

int main()
{
    // 下列各个情况推导失败：
    f<A>(0); // 不含成员 Y
    f<B>(0); // B 的 Y 成员不是类型
    g<C>(0); // C 的 N 成员不是非类型
    h<D>(0); // D 的 TT 成员不是模板
 
    // 下列各个情况推导成功：
    f<B1>(0); 
    g<C1>(0); 
    h<D1>(0);
}
// 未完成：需要演示重载决议，而不只是失败
```

- 试图创建指向引用的指针
- 试图创建到 void 的引用
- 试图创建指向 T 的成员的指针，其中 T 不是类类型：
```c++
template<typename T>
class is_class
{
    typedef char yes[1];
    typedef char no[2];
 
    template<typename C>
    static yes& test(int C::*); // 如果 C 是类类型就选择它
 
    template<typename C>
    static no& test(...);       // 否则选择它
public:
    static bool const value = sizeof(test<T>(0)) == sizeof(yes);
};
```

- 试图将非法类型给予非类型模板形参：
```c++
template<class T, T>
struct S {};
 
template<class T>
int f(S<T, T()>*);
 
struct X {};
int i0 = f<X>(0);
// 未完成：需要演示重载决议，而不只是失败
```

- 试图在以下语境中进行非法转换：

- 模板实参表达式
- 函数声明中使用的表达式：
```c++

template<class T, T*> int f(int);
int i2 = f<int,1>(0); // 不能将 1 转换为 int*
// 未完成：需要演示重载决议，而非仅是失败
```

- 试图创建形参类型为 void 的函数类型
- 试图创建返回数组类型或函数类型的函数类型

## 表达式 SFINAE


C++11 前，只有在类型中使用的常量表达式（例如数组边界）才要求被当做 SFINAE（而非硬错误）。(C++11 前)
下列表达式错误是 SFINAE 错误：
- 模板形参类型中使用的非良构表达式
- 函数类型中使用的非良构表达式
```c++

struct X {};
struct Y { Y(X){} }; // X 可以转换到 Y
template<class T>
auto f(T t1, T t2) -> decltype(t1 + t2); // 重载 #1
X f(Y, Y);                               // 重载 #2
X x1, x2;
// 推导在 #1 上失败（表达式 x1+x2 非良构）
X x3 = f(x1, x2);  
// 重载集中只有 #2，调用它(C++11 起)
```

## 部分特化中的 SFINAE

在确定一个类或变量 (C++14 起)模板的特化是由 部分特化 还是主模板生成的时候也会出现推导与替换。在这种确定期间，部分特化的替换失败不会被当作硬错误，而是像涉及函数模板的重载决议中一样使得对应的部分特化声明被忽略。

```c++
// 主模板处理无法引用的类型：
template<class T, class = void>
struct reference_traits{
    using add_lref = T;
    using add_rref = T;
};
 
// 特化识别可以引用的类型：
template<class T>
struct reference_traits<T, std::void_t<T&>>{
    using add_lref = T&;
    using add_rref = T&&;
};

template<class T>
using add_lvalue_reference_t = typename reference_traits<T>::add_lref;
 
template<class T>
using add_rvalue_reference_t = typename reference_traits<T>::add_rref;
```
# 库支持

标准库组件 [std::enable_if]允许创建替换失败，以基于某个在编译时求值的条件来启用或禁用特定的重载。

另外，许多[类型特征]在适合的编译器扩展不可用时必须以 SFINAE 实现。(C++11 起)

标准库组件 [std::void_t] 是另一个简化部分特化 SFINAE 的应用的工具元函数。(C++17 起)

# 替代方案

只要可用，[标签派发]、[`if constexpr`] (C++17 起)以及[概念] (C++20 起)通常都比直接使用 SFINAE 更好。

如果只想要条件性的编译时错误，[`static_assert`]通常比 SFINAE 更适合。(C++11 起)

## 示例

一种常见手法，是在返回类型上使用表达式 SFINAE，其中表达式使用逗号运算符，其左子表达式是所检验的（转型到 void 以确保不会选择返回类型上的用户定义逗号运算符），而右子表达式具有期望函数返回的类型。

运行此代码
```c++
#include <iostream>
 
// 如果 C 是类或者类的引用类型且 F 是指向 C 的成员函数的指针，
// 那么这个重载会被添加到重载集
template<class C, class F>
auto test(C c, F f) -> decltype((void)(c.*f)(), void())
{
    std::cout << "(1) 调用了类/类引用重载\n";
}
 
// 如果 C 是类的指针类型且 F 是指向 C 的成员函数的指针，
// 那么这个重载会被添加到重载集
template<class C, class F>
auto test(C c, F f) -> decltype((void)((c->*f)()), void())
{
    std::cout << "(2) 调用了指针重载\n";
}
 
// 此重载始终在重载集中：
// 省略号形参对于重载决议具有最低等级
void test(...)
{
    std::cout << "(3) 调用了保底重载\n";
}
 
int main()
{
    struct X { void f() {} };
    X x;
    X& rx = x;
    test(x, &X::f);  // (1)
    test(rx, &X::f); // (1), 创建了 x 的副本
    test(&x, &X::f); // (2)
    test(42, 1337);  // (3)
}
```

输出：
```
(1) 调用了类/类引用重载
(1) 调用了类/类引用重载
(2) 调用了指针重载
(3) 调用了保底重载
```

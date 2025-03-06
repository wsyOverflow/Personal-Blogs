---
typora-copy-images-to: ..\..\images\draft\C++ Reflection\${filename}.assets
typora-root-url: ..\..\
title: Reflection without Reflection (TS)
keywords: C++, Reflection
categories:
- [C++, Reflection]
mathjax: true
---

# 1 Introduction

反射特性还不会由 C++23 标准支持，这篇用来讲解如何使用目前的C++标准来实现反射，会涉及哪些技术。

# 2 Question？

 ## 2.1 是否为指针类型？

```c++
std::cout << std::is_pointer_v<decltype(a)> << std::endl;
```

### 2.1.1 Reflection technique #1: Template/function specialization

```c++
std::cout << std::is_array_v<int[5]> << std::endl;
```

### 2.1.2 Reflection technique #2: template argument deduction

```c++
template <typename T, std::size_t N>
constexpr std::integral_constant<std::size_t, N> array_length_helper(T(&&)[N]);

template <typename T>
static constexpr auto array_length = decltype(array_length_helper(std::declval<T>()))::value;

int a[5];
std::cout << array_length<decltype(a)> << std::endl;
```

## 2.2 是否为多态类型？

### 2.2.1 Reflection technique #3: SFINAE

SFINAE(Substitution Failure Is Not An Error)发生在模板实例化之前。如果模板参数替换过程中发生了错误，例如类型表达式或语句无效，编译器不会报错，而是选择不为该模板生成实例化代码（跳过该重载版本），如重载决议中的挑选最佳模板函数过程。

Use SFINAE to check if an expression would generate a compiler-error

```c++
template <typename T>
std::true_type check_expression(decltype(sizeof(      ))); // 括号内填写要检查的expression
template <typename T>
std::false_type check_expression(...);
```

Find an expression that is only ever well-formed when **T** is polymorphic?

借助于 dynamic_cast 的转换，当目标转换类型为 `void*` 时，只有多态类型是有效的转换

![image-20240128133452901](/images/draft/C++ Reflection/Reflection without Reflection (TS).assets/image-20240128133452901.png)

```c++
template <typename T>
std::true_type check_expression(decltype(sizeof(dynamic_cast<void*>(std::declval<T*>()))));
```

对于非多态类型上述表达式变换是不合法的，因此可以基于此判断是否为多态类型

```c++
template <typename T>
static constexpr auto is_polymorphic = (decltype(check_expression<T>(0))::value != 0);
```

## 2.3 是否为struct、class类型？

编译器支持的trait

```c++
std::is_cast<T>
```

## 2.4 编译时返回enum name

例如实现下面的功能

```c++
std::cout << enum_name<Color::red>() << std::endl;
```

### 2.4.1 Reflection technique #4: `std::source_location::function_name`

```c++
template <auto enumValue>
constexpr std::string_view enum_name() {
    return std::source_location::current().function_name();
}

std::cout << enum_name<Color::red>() << std::endl;
// Prints:
// std::string_view enum_name() [enumValue = Color::red]
```

上面代码在编译期生成 enum_name 的输出信息，可以根据该信息的模式，对字符串进行操作，提出所需信息

```c++
template <auto enumValue>
constexpr std::string_view enum_name() {
    std::string_view thisFunctioName(std::source_location::current().function_name());
    auto enum_start = thisFunctionName.substr(thisFunctionName.find('=') + 2);
    auto enum_value = enum_start.substr(enum_start.find("::") + 2);
    return enum_value.substr(0, enum_value.size() - 1);
}

std::cout << enum_name<Color::red>() << std::endl;
// Prints:
// red
```

但这种编译信息可能对于不同编译器会有不同格式，例如MSVC不会将enum-value包含在 std::source_location 中，而是使用 `__FUNCSING__` 宏。可以使用开源库 [magic_enum](https://github.com/Neargye/magic_enum) 来处理。

## 2.5 POD class/struct成员的names、types ?

**Plain Old Data (POD)** 结构体指的是只包含数据，并且没有自定义构造、析构函数，也称为 aggregate class type。

### 2.5.1 成员数量

Struct reflection: counting the number of members

- Check if a struct has at least N members
- Assume struct only has integral type members

### 2.5.2 `std::tuple<...>` 成员类型元组

#### 2.5.2.1 Reflection technique #5: structured bindings

Structured Bindings can introduce a Pack: https://wg21.link/p1061

https://godbolt.org/z/T47PaWq6E

https://www.youtube.com/watch?v=abdeAew3gmQ



### 2.5.3 成员名字(clang-only)

#### 2.5.3.1 Reflection Technique #6: clang’s `__builtin_dump_struct` 



## 2.6 是否使用 tab 或空格

借助于 `std::source_location`

```c++
static constexpr auto is_using_tabs = std::source_location::current().column() == 2;
```

如果不使用 `std::source_location`

https://godbolt.org/z/4GvfoEMTT



# 3 Summary

所使用到的技术有：

- STL’s type-traits
- Template/function specialization
- Template argument deduction
- SFINAE
- `std::source_location` (or `__PRETTY_FUNCTION__` or `__FUNCSIG__`)
- Structured bindings
- (Clang’s `__builtin_dump_struct`)

---
description: 一些C++11/14/17小点
---

# 😅 Miscellaneous

> 首先要放最全面的学习资料：
>
> ****[**本人最喜欢的C++介绍频道**](https://www.youtube.com/user/CppCon/videos)****
>
> 官方文档：
>
> [https://en.cppreference.com/w/cpp/compiler\_support](https://en.cppreference.com/w/cpp/compiler\_support)
>
> [https://en.cppreference.com/w](https://en.cppreference.com/w)

### `long long int`

`long long int` 并不是 C++11 最先引入的，其实早在 C99， `long long int` 就已经被纳入 C 标准中，所以大部分的编译器早已支持。&#x20;

C++11 的工作则是正式把它纳入标准库， 规定了一个 `long long int` 类型至少具备 64 位的比特数。

### noexcept 的修饰和操作

C++ 相比于 C 的一大优势就在于 C++ 本身就定义了一套完整的异常处理机制。 然而在 C++11 之前，几乎没有人去使用在函数名后书写异常声明表达式， 从 C++11 开始，这套机制被弃用，所以我们不去讨论也不去介绍以前这套机制是如何工作如何使用， 你更不应该主动去了解它。

C++11 将**异常的声明**简化为以下两种情况：

1. 函数可能抛出任何异常
2. 函数不能抛出任何异常

并使用 `noexcept` 对这两种行为进行限制，例如：

`void may_throw(); // 可能抛出异常`&#x20;

`void no_throw() noexcept; // 不可能抛出异常`

使用 `noexcept` 修饰过的函数如果抛出异常，编译器会使用 `std::terminate()` 来立即终止程序运行。

`noexcept` 还能够做操作符，用于**操作一个表达式**，当表达式无异常时，返回 `true`，否则返回 `false`。

```cpp
#include <iostream>  
void may_throw() {  
 throw true;  
}  
auto non_block_throw = []{  
 may_throw();  
};  
void no_throw() noexcept {  
 return;  
}

auto block_throw = []() noexcept {  
 no_throw();  
};  
int main()  
{  
 std::cout << std::boolalpha  
 << "may_throw() noexcept? " << noexcept(may_throw()) << std::endl  
 << "no_throw() noexcept? " << noexcept(no_throw()) << std::endl  
 << "lmay_throw() noexcept? " << noexcept(non_block_throw()) << std::endl  
 << "lno_throw() noexcept? " << noexcept(block_throw()) << std::endl;  
 return 0;  
}
```

`noexcept` 修饰完一个函数之后能够起到**封锁异常扩散**的功效，如果内部产生异常，外部也不会触发。例如：

```cpp
try {  
    may_throw();  
} catch (...) {  
    std::cout << "捕获异常, 来自 may_throw()" << std::endl;  
}  
try {  
    non_block_throw();  
} catch (...) {  
    std::cout << "捕获异常, 来自 non_block_throw()" << std::endl;  
}  
try {  
    block_throw();  // 函数被noexcept修饰
} catch (...) {  
    std::cout << "捕获异常, 来自 block_throw()" << std::endl;  
}  
```

最终输出为：

```
捕获异常, 来自 may_throw()  
捕获异常, 来自 non_block_throw()  
```

### 字面量

#### 原始字符串字面量

传统 C++ 里面要编写一个充满特殊字符的字符串其实是非常痛苦的一件事情， 比如一个包含 HTML 本体的字符串需要添加大量的转义符， 例如一个Windows 上的文件路径经常会：`C:\\File\\To\\Path`。

C++11 提供了原始字符串字面量的写法，可以在一个字符串前方使用 <mark style="color:red;">**`R`**</mark> 来修饰这个字符串， 同时，将原始字符串使用括号包裹，例如：

```cpp
#include <iostream>  
#include <string>  
int main() {  
    std::string str = R"(C:\File\To\Path)";  
    std::cout << str << std::endl;  
    return 0;  
}  
```

#### 自定义字面量

C++11 引进了自定义字面量的能力，通过**重载双引号后缀运算符**实现：

字符串字面量自定义必须设置如下的**参数列表**

```cpp
std::string operator"" _wow1(const char *wow1, size_t len) {  
    return std::string(wow1) + "woooooooooow, amazing";  
}  
std::string operator"" _wow2 (unsigned long long i) {  
    return std::to_string(i) + "woooooooooow, amazing";  
}  
int main() {  
    auto str = "abc"_wow1;  
    auto num = 1_wow2;  
    std::cout << str << std::endl;  
    std::cout << num << std::endl;  
    return 0;  
}  
```

自定义字面量支持四种字面量：

1. **整型字面量**：重载时必须使用 `unsigned long long`、`const char *`、模板字面量算符参数，在上面的代码中使用的是前者；
2. **浮点型字面量**：重载时必须使用 `long double`、`const char *`、模板字面量算符；
3. **字符串字面量**：必须使用 `(const char *, size_t)` 形式的参数表；
4. **字符字面量**：参数只能是 `char`, `wchar_t`, `char16_t`, `char32_t` 这几种类型。

### 内存对齐

C++ 11 引入了两个新的关键字 **`alignof`** 和 **`alignas`** 来支持对内存对齐进行控制。 `alignof` 关键字能够获得一个与平台相关的 `std::size_t` 类型的值，用于**查询**该平台的对齐方式。 当然我们有时候并不满足于此，甚至希望自定定义结构的对齐方式，同样，C++ 11 还引入了 `alignas` 来**重新修饰**某个结构的对齐方式。我们来看两个例子：

```cpp
#include <iostream>  
struct Storage {  
    char      a;  
    int       b;  
    double    c;  
    long long d;  
};  
struct alignas(std::max_align_t) AlignasStorage {  
    char      a;  
    int       b;  
    double    c;  
    long long d;  
};  
int main() {  
    std::cout << alignof(Storage) << std::endl;  
    std::cout << alignof(AlignasStorage) << std::endl;  
    return 0;  
}  
```

其中 `std::max_align_t` 要求每个标量类型的对齐方式严格一样，因此它几乎是最大标量没有差异， 进而大部分平台上得到的结果为 `long double`，因此我们这里得到的 `AlignasStorage` 的对齐要求是 8 或 16。

### C艹11 的扬弃

*   不再允许字符串字面值常量赋值给一个 `char *`。如果需要用字符串字面值常量赋值和初始化一个 `char *`，应该使用 `const char *` 或者 `auto`。

    例如：

    ```
    char *str = "hello world!";  // 将出现弃用警告
    ```
* C++98 异常说明、 `unexpected_handler`、`set_unexpected()` 等相关特性被弃用，应该使用 `noexcept`。
* `auto_ptr` 被弃用。
* `register` 关键字被弃用，可以使用但不再具备任何实际含义。
* `bool` 类型的 `++` 操作被弃用。
* 如果一个类有析构函数，为其生成拷贝构造函数和拷贝赋值运算符的特性被弃用了。
* C 语言风格的类型转换被弃用（即在变量前使用 `(convert_type)`），应该使用 `static_cast`、`reinterpret_cast`、`const_cast` 来进行类型转换。
* 特别地，在最新的 C++17 标准中弃用了一些可以使用的 C 标准库，例如 `<ccomplex>`、`<cstdalign>`、`<cstdbool>` 与 `<ctgmath>` 等

### 常量

#### nullptr

`nullptr` 出现的目的是为了替代 `NULL`。在某种意义上来说，传统 C++ 会把 `NULL`、`0` 视为同一种东西，这取决于编译器如何定义 `NULL`，有些编译器会将 `NULL` 定义为 `((void*)0)`，有些则会直接将其定义为 `0`。

C++ **不允许**直接将 `void *` 隐式转换到其他类型。但如果编译器尝试把 `NULL` 定义为 `((void*)0)`，那么在下面这句代码中：`char *ch = NULL;` 中没有了 `void *` 隐式转换的 C++ 只好将 `NULL` 定义为 `0`。而这依然会产生新的问题，将 `NULL` 定义成 `0` 将导致 `C++` 中重载特性发生混乱。

比如 `void f(char*)` 和 `void f(int)`，`f(NULL)` 会调用后者。

C++11 引入了 `nullptr` 关键字，专门用来区分空指针、`0`。而 `nullptr` 的类型为 `nullptr_t`，能够隐式的转换为任何指针或成员指针的类型，也能和他们进行相等或者不等的比较。

#### constexpr

如果编译器能够在编译时就把 1 + 2 类似的常量表达式直接优化并植入到程序运行时，将能增加程序的性能。

这里有个小🌰

```cpp
#include <iostream>
#define LEN 10 // const 

int len_foo() {
    int i = 2; 
    return i;
}

constexpr int len_foo_constexpr() {
    return 5; // 5 : const
}

constexpr int fibonacci(const int n) {
    return n == 1 || n == 2 ? 1 : fibonacci(n - 1) + fibonacci(n - 2);
}

int main() {
    char arr_1[10];                      // 合法
    char arr_2[LEN];                     // 合法

    int len = 10;
    // char arr_3[len];                  // 非法！！！！！

    const int len_2 = len + 1;
    constexpr int len_2_constexpr = 1 + 2 + 3;
    // char arr_4[len_2];                // 非法！！！！！
    char arr_4[len_2_constexpr];         // 合法

    // char arr_5[len_foo()+5];          // 非法！！！！！
    char arr_6[len_foo_constexpr() + 1]; // 合法 🐶

    std::cout << fibonacci(10) << std::endl;
    // 1, 1, 2, 3, 5, 8, 13, 21, 34, 55
    std::cout << fibonacci(10) << std::endl;
    return 0;
}
```

上面的例子中，`char arr_4[len_2]` 可能比较令人困惑，因为 `len_2` 已经被定义为了常量。

**为什么 `char arr_4[len_2]` 仍然是非法的呢？**

这是因为 **C++ 标准中数组的长度必须是一个常量表达式**，而对于 `len_2` 而言，这是一个 `const` 常数，而不是一个常量表达式，因此（即便这种行为在大部分编译器中都支持，但是）它是一个非法的行为，我们需要使用 C++11 引入的 `constexpr` 特性来解决这个问题。

**『** 下面说说常量和常量表达式：

常量是固定值，在程序执行期间不会改变。**常量表达式** 定义能在**编译**时求值的表达式。这种表达式能用做非类型模板实参、数组大小，并用于其他要求常量表达式的语境。

但是常量不是常量表达式，只有**用常量表达式初始化的常量**，才能成为常量表达式，**用非常量表达式初始化的常量**比如上文的`len_2`仅仅是常量。如果常量的初始值不是常量表达式,则该常量不是常量表达式。

* 定义一个常量变量，只需要把`const`放到变量类型的前面**或者后面**就可以了
* 定义`const`变量而不初始化会导致编译错误
* 常量变量的初始化可以来自于普通变量
*   C++有两种类型地常量：编译时间常量和运行时间常量：

    运行时间常量表示只有在程序运行的时候才会初始化，编译时间常量是在编译时间就被初始化了；

    在大多数场景下，编译时间常量还是运行时间常量是可以互换的。

    然而在一些特殊场景下，C++需要一些编译时间常量，为了作出区分，C++11新定义了一个关键字`constexpr`定义编译时间常量。

再顺便提一嘴符号常量:

在多数程序里，一个符号常量需要从头用到尾。这种包括数学或者物理上的一些常数。与其每次都定义一遍，比如直接在一个核心位置定义一次，其他地方随时使用。这样，如果你需要改动的话，只需要改动一个地方就OK了。

在C++里有很多种方式来实现这一点，但是下面的是最容易的：

* 创建一个头文件来包含这些常量
* 在这个头文件里定义一个命名空间
* 在你的命名空间里加上你的所有常量
* 当你使用的时候#include你的头文件

```cpp
#ifndef CONSTANTS_H
#define CONSTANTS_H
 
// define your own namespace to hold constants
namespace constants {
    constexpr double pi(3.14159);
    constexpr double avogadro(6.0221413e23);
    constexpr double my_gravity(9.2); // m/s^2 -- gravity is light on this planet
    // ... other related constants
}
#endif
```

使用范围操作符`::`来引用

```objectivec
#include "constants.h"
double circumference = 2 * radius * constants::pi;
```

如果你同时有物理学的常量和app的调优值，你可以弄两个文件。一个是永远不变的物理常量，另一个是可能有变化的调优值。这样物理常量你可以到处使用了。**』**

接下来回到上面的函数：

而对于 `arr_5` 来说，C++98 之前的编译器无法得知 `len_foo()` 在运行期实际上是返回一个常数，这也就导致了非法的产生。

C++11 提供了 `constexpr` 让用户**显式的声明函数或对象构造函数**在编译期会成为常量表达式，这个关键字明确的告诉编译器应该去验证 `len_foo` 在编译期就应该是一个常量表达式。

此外，`constexpr` 修饰的函数可以使用递归。

从 C++**14** 开始，`constexpr` 函数可以**在内部使用局部变量、循环和分支等简单语句**，例如下面的代码在 C++**11** 的标准下是**不能**够通过编译的：、

```cpp
constexpr int fibonacci(const int n) {
    if(n == 1) return 1;
    if(n == 2) return 1;
    return fibonacci(n-1) + fibonacci(n-2);
}
```

为此，我们可以写出上面的简化的版本来使得函数从 C++11 开始即可用。

### 变量

在传统 C++ 中，变量的声明虽然能够位于任何位置，甚至于 `for` 语句内能够声明一个临时变量 `int`，但始终没有办法在 `if` 和 `switch` 语句中声明一个临时的变量。例如：

```cpp
int main() {
    std::vector<int> vec = {1, 2, 3, 4};

    // 在 c++17 之前
    const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 2);
    if (itr != vec.end()) {
        *itr = 3;
    }

    // 需要重新定义一个新的变量
    const std::vector<int>::iterator itr2 = std::find(vec.begin(), vec.end(), 3);
    if (itr2 != vec.end()) {
        *itr2 = 4;
    }

    // 将输出 1, 4, 3, 4
    for (std::vector<int>::iterator element = vec.begin(); element != vec.end(); 
        ++element)
        std::cout << *element << std::endl;
}
```

在上面的代码中，我们可以看到 `itr` 这一变量是定义在整个 `main()` 的作用域内的，这导致当我们需要再次遍历整个 `std::vectors` 时，需要重新命名另一个变量。

**C++17 消除了这一限制，使得我们可以在 `if`（或 `switch`）中完成这一操作：**

```cpp
// 将临时变量放到 if 语句内
if (const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 3);
    itr != vec.end()) {
    *itr = 4;
}
```

#### 初始化列表

最常见的就是在对象进行初始化时进行使用

传统 C++ 中，不同的对象有着不同的初始化方法，例如普通数组、 POD （**P**lain **O**ld **D**ata，即没有构造、析构和虚函数的类或结构体） 类型都可以使用 `{}` 进行初始化，也就是我们所说的初始化列表。 而对于类对象的初始化，要么需要通过拷贝构造、要么就需要使用 `()` 进行。 这些不同方法都针对各自对象，不能通用。

为解决这个问题，C++11 首先把初始化列表的概念绑定到类型上，称其为 <mark style="color:green;background-color:green;">`std::initializer_list`</mark>，允许构造函数或其他函数像参数一样使用初始化列表，这就为**类对象的初始化与普通数组和 POD 的初始化**方法提供了统一的桥梁，例如：

```cpp
#include <initializer_list>
#include <vector>
#include <iostream>

class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
        for (std::initialozer_list<int>::iterator it = list.begin();
             it != list.end(); ++it) {
            vec.push_back(*it);
        }
    }
};
int main() {
    // after C++11
    MagicFoo magicFoo = {1, 2, 3, 4, 5};

    std::cout << "magicFoo: ";
    for (std::vector<int>::iterator it = magicFoo.vec.begin(); 
        it != magicFoo.vec.end(); ++it) {
        std::cout << *it << std::endl;
    }
}
```

这种构造函数被叫做初始化列表构造函数，具有这种构造函数的类型将在初始化时被特殊关照。

初始化列表除了用在对象构造上，还能将其作为普通函数的形参，例如：

```cpp
public:
    void foo(std::initializer_list<int> list) {
        for (std::initializer_list<int>::iterator it = list.begin();
            it != list.end(); ++it) {
            vec.push_back(*it);
        }
    }

magicFoo.foo({6,7,8,9});
```

其次，C++11 还提供了统一的语法来初始化任意的对象，例如：

```
Foo foo2 {3, 4};
```

#### 结构化绑定

结构化绑定提供了类似其他语言中提供的多返回值的功能。C++11 新增了 `std::tuple` 容器用于构造一个元组，进而囊括多个返回值。但缺陷是，C++11/14 并没有提供一种简单的方法直接从元组中拿到并定义元组中的元素，尽管我们可以使用 `std::tie` 对元组进行拆包，但我们依然必须非常清楚这个元组包含多少个对象，各个对象是什么类型，非常麻烦。

C++**17** 完善了这一设定，给出的结构化绑定可以让我们写出这样的代码：

```cpp
#include <iostream>
#include <tuple>

std::tuple<int, double, std::string> f() {
    return std::make_tuple(1, 2.3, "456");
}

int main() {
    auto [x, y, z] = f();
    std::cout << x << ", " << y << ", " << z << std::endl;
    return 0;
}
```

### 类型推导

C++11 引入了 `auto` 和 `decltype` 这两个关键字实现了类型推导，让编译器来操心变量的类型。

这使得 C++ 也具有了和其他现代编程语言一样，某种意义上提供了无需操心变量类型的使用习惯。

#### auto

`auto` 在很早以前就已经进入了 C++，但是他始终作为一个存储类型的指示符存在，与 `register` 并存。在传统 C++ 中，如果一个变量没有声明为 `register` 变量，将自动被视为一个 `auto` 变量。而随着 `register` 被弃用（在 C++17 中作为保留关键字，以后使用，目前不具备实际意义），对 `auto` 的语义变更也就非常自然了。

使用 `auto` 进行类型推导的一个最为常见而且显著的例子就是迭代器。你应该在前面的小节里看到了传统 C++ 中冗长的迭代写法：

```cpp
// 在 C++11 之前
// 由于 cbegin() 将返回 vector<int>::const_iterator
// 所以 itr 也应该是 vector<int>::const_iterator 类型
for(vector<int>::const_iterator it = vec.cbegin(); itr != vec.cend(); ++it)
```

而有了 `auto` 之后可以：

```cpp
class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
        // 从 C++11 起, 使用 auto 关键字进行类型推导
        for (auto it = list.begin(); it != list.end(); ++it) {
            vec.push_back(*it);
        }
    }
};
int main() {
    MagicFoo magicFoo = {1, 2, 3, 4, 5};
    std::cout << "magicFoo: ";
    for (auto it = magicFoo.vec.begin(); it != magicFoo.vec.end(); ++it) {
        std::cout << *it << ", ";
    }
    std::cout << std::endl;
    return 0;
}
```

一些其他的常见用法：

```cpp
auto i = 5;              // i 被推导为 int
auto arr = new auto(10); // arr 被推导为 int *
```

从 **C++ 20** 起，`auto` 甚至能用于函数传参，考虑下面的例子：

```cpp
int add(auto x, auto y) {
    return x+y;
}

auto i = 5; // 被推导为 int
auto j = 6; // 被推导为 int
std::cout << add(i, j) << std::endl;

```

**注意**：`auto` 还不能用于推导数组类型：

```cpp
auto auto_arr2[10] = {arr}; // 错误, 无法推导数组元素类型
```

#### decltype(auto)

`decltype(auto)` 是 C++14 开始提供的一个略微复杂的用法。

简单来说，`decltype(auto)` 主要用于对转发函数或封装的返回类型进行推导，它使我们无需显式的指定 `decltype` 的参数表达式。考虑看下面的例子，当我们需要对下面两个函数进行封装时：

```cpp
std::string  lookup1();
std::string& lookup2();
```

在 C++11 中，封装实现是如下形式：

```cpp
std::string look_up_a_string_1() {
    return lookup1();
}
std::string& look_up_a_string_2() {
    return lookup2();
}
```

而有了 `decltype(auto)`，我们可以让编译器完成这一件烦人的参数转发：

```cpp
decltype(auto) look_up_a_string_1() {
    return lookup1();
}
decltype(auto) look_up_a_string_2() {
    return lookup2();
}
```

### 控制流

#### if constexpr

&#x20;C++11 引入了 `constexpr` 关键字，它将表达式或函数编译为常量结果。一个很自然的想法是，如果我们把这一特性引入到条件判断中去，让代码在编译时就完成分支判断，岂不是能让程序效率更高？

**C++17** 将 `constexpr` 这个关键字引入到 `if` 语句中，**允许在代码中声明常量表达式的判断条件**，考虑下面的代码：

```cpp
template<typename T>
auto print_type_info(const T& t) {
    if constexpr (std::is_integral<T>::value) {
        return t + 1;
    } else {
         return t + 0.01;
    }
}
int main() {
    std::cout << print_type_info(5) << std::endl;
    std::cout << print_type_info(3.14) << std::endl;
}

// 在编译时，实际代码就会表现为如下：
/*
int print_type_info(const int& t) {
    return t + 1;
}
double print_type_info(const double& t) {
    return t + 0.001;
}
int main() {
    std::cout << print_type_info(5) << std::endl;
    std::cout << print_type_info(3.14) << std::endl;
}
*/
```

#### 区间for迭代

**C++11** 引入了基于范围的迭代写法

```cpp
int main() {
    std::vector<int> vec = {1, 2, 3, 4};
    if (auto itr = std::find(vec.begin(), vec.end(), 3); itr != vec.end()) {
        *itr = 4;
    }
    for (auto element : vec)
        std::cout << element << std::endl; // read only
    for (auto &element : vec) {
        element += 1;                      // writeable
    }    
}
```

### template

#### 外部模板

传统 C++ 中，模板只有在使用时才会被编译器实例化。

换句话说，只要在每个编译单元（文件）中编译的代码中遇到了被完整定义的模板，都会实例化。这就产生了重复实例化而导致的编译时间的增加。并且，我们没有办法通知编译器不要触发模板的实例化。

为此，**C++11** 引入了外部模板，扩充了原来的强制编译器在特定位置实例化模板的语法，使我们能够显式的通知编译器**何时进行模板的实例化**：

```cpp
template class std::vector<bool>; // 强行实例化
extern template class std::vector<double>; // 不在该当前编译文件中实例化模板 
```

#### 尖括号 ">"

传统 C++ 的编译器中，`>>`一律被当做右移运算符来进行处理。但实际上我们很容易就写出了嵌套模板的代码：`std::vector<std::vector<int>> matrix;` 在传统C++中不能编译

C++11 开始，连续的右尖括号将变得合法，并且能够顺利通过编译。甚至于像下面这种写法都能够通过编译：

```cpp
template<bool T>
class MType {
    bool m = T;
}
int main() {
 ...
 std::vector<MType(1>2)>> m; // 合法, 但不建议写出这样的代码
```

#### 类型别名模板

仔细体会这句话：**模板是用来产生类型的。**

在传统 C++ 中，`typedef` 可以为类型定义一个新的名称，但是却没有办法为模板定义一个新的名称。因为，模板不是类型。例如：

```cpp
template<typename T, typename U>
class MagicType {
public:
    T dark;
    U magic;
};

// 不合法
template<typename T>
typedef MagicType<std::vector<T>, std::string> FakeDarkMagic;
```

**C++11** 使用 `using` 引入了下面这种形式的写法，并且同时支持对传统 `typedef` 相同的功效：

通常我们使用 `typedef` 定义别名的语法是：`typedef 原名称 新名称;`，但是对函数指针等别名的定义语法却不相同，这通常给直接阅读造成了一定程度的困难。

```cpp
template<typename T, typename U>
class MagicType {
public:
    T dark;
    U magic;
};

// 1st
typedef int (*process)(void *);
using NewProcess = int(*)(void *);

// 2nd
template<typename T>
using TrueMagic = MagicType<std::vector<T>, std::string>;
int main() {
    TrueMagic<bool> you;
}    
```

#### 变长参数模板

在 C++11 之前，无论是类模板还是函数模板，都只能按其指定的样子， 接受一组固定数量的模板参数；

而 C++11 加入了新的表示方法， 允许任意个数、任意类别的模板参数，同时也不需要在定义时将参数的个数固定。

```cpp
template<typename... Ts> class Magic;
```

模板类 Magic 的对象，能够接受不受限制个数的 `typename` 作为模板的形式参数，例如下面的定义：

```cpp
class Magic<int, 
            std::vector<int>, 
            std::map<std::string, std::vector<int>>> darkMagic;
```

既然是任意形式，所以个数为 `0` 的模板参数也是可以的：`class Magic<> nothing;`。

如果不希望产生的模板参数个数为 `0`，可以手动的定义至少一个模板参数：

```cpp
template<typename Require, typename... Args> class Magic2;
```

变长参数模板也能被直接调整到到模板函数上。

传统 C 中的 `printf` 函数， 虽然也能达成不定个数的形参的调用，但其并非类别安全。 而 C++11 除了能定义类别安全的变长参数函数外， 还可以使类似 `printf` 的函数能自然地处理非自带类别的对象。 除了在模板参数中能使用 `...` 表示不定长模板参数外， **函数参数**也使用同样的表示法代表不定长参数， 这也就为我们简单编写变长参数函数提供了便捷的手段，例如：

```cpp
template<typename... Args> 
void printf(const std::string &str, Arg... args);
```

那么我们定义了变长的模板参数，如何对参数进行解包呢？

首先，我们可以使用 `sizeof...` 来计算参数的个数，：

```cpp
template<typename... Args>
void magic(Args... args) {
    std::cout << sizeof...(args) << std::endl;
}
```

我们可以传递任意个参数给 `magic` 函数：

其次，对参数进行解包，到目前为止还没有一种简单的方法能够处理参数包，但有两种经典的处理手法：

**1. 递归模板函数**

递归是非常容易想到的一种手段，也是最经典的处理方法。这种方法不断**递归地向函数传递模板参数**，进而达到递归遍历所有模板参数的目的：

```cpp
template<typename T0>
void printf0(T0 value) { // 单参数——递归最后一步
    std::cout << value << std::endl;
}
template<typename T, typename... Ts>
void printf0(T value, Ts... args) { // 多参数——递归开始
    std::cout << value << std::endl;
    printf0(args...);
}
int main() {
    printf0(1, 2, "123", 1.1);
    return 0;
}
```

**2. 变参模板展开**

你应该感受到了这很繁琐，在 C++17 中增加了变参模板展开的支持，于是你可以在一个函数中完成 `printf` 的编写：

```cpp
template<typename T0, typename... Ts>
void printf1(T0 t0, Ts... t) {
    std::cout << t0 << std::endl;
    if constexpr (sizeof...(t) > 0) {
        printf1(t...);
    }
}
```

事实上，有时候我们虽然使用了变参模板，却不一定需要对参数做逐个遍历，

我们可以利用 **`std::bind`** 及完美转发等特性实现对函数和参数的绑定，从而达到成功调用的目的。

**3. 初始化列表展开**

递归模板函数是一种标准的做法，但缺点显而易见的在于必须定义一个终止递归的函数。

这里介绍一种使用初始化列表展开的黑魔法：

```cpp
template<typename T, typename... Ts>
auto printf2(T value, Ts... args) {
    std::cout << value << std::endl;
    (void) std::initializer_list<T> { ( [&args] {
        std::cout << args << std::endl;
    }(), value)...};
}
```

在这个代码中，额外使用了 C++11 中提供的初始化列表以及 Lambda 表达式的特性（下一节中将提到）。

通过初始化列表，`(lambda 表达式, value)...` 将会被展开。

由于逗号表达式的出现，首先会执行前面的 lambda 表达式，完成参数的输出。&#x20;

为了避免编译器警告，我们可以将 `std::initializer_list` 显式的转为 `void`。

#### 折叠表达式

**C++ 17** 中将变长参数这种特性进一步带给了表达式，考虑下面这个例子：

```cpp
template<typename ... T>
auto sum(T ... t) {
    return (t + ...);
}
int main() {
    std::cout << sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) << std::endl;
}
```

#### 非类型模板参数推导

前面我们主要提及的是模板参数的一种形式：类型模板参数。

```cpp
template <typename T, typename U>
auto add(T t, U u) {
    return t+u;
}
```

其中模板的参数 `T` 和 `U` 为具体的类型。&#x20;

但还有一种常见模板参数形式可以让**不同字面量成为模板参数**，即非类型模板参数：

```cpp
template<typename T, int BufSize>
class buffer_t {
public:
    T& alloc();
    void free(T& item);
private:
    T data[BufSize];
}
buffer_t<int, 100> buf;
```

在这种模板参数形式下，我们可以将 `100` 作为模板的参数进行传递。&#x20;

在 **C++11** 引入了类型推导这一特性后，我们会很自然的问，既然此处的模板参数 以具体的字面量进行传递，能否让编译器辅助我们进行类型推导， 通过使用占位符 `auto` 从而不再需要明确指明类型？&#x20;

幸运的是，**C++17** 引入了这一特性，我们的确可以 `auto` 关键字，让编译器辅助完成具体类型的推导， 例如：

```cpp
template <auto value> 
void foo() {
    std::cout << value << std::endl;
    return;
}

int main() {
    foo<10>();  // value 被推导为 int 类型
}
```

### 面向对象

#### 委托构造

C++11 引入了委托构造的概念，这使得可以**在同一个类中一个构造函数调用另一个构造函数**，从而达到简化代码的目的：

```cpp
class Base {
public:
    int value1;
    int value2;
    Base() {
        value1 = 1;
    }
    Base(int value) : Base() { // 委托 Base() 构造函数
        value2 = value;
    }
};
```

#### 继承构造

传统 C++ 中，构造函数如果需要继承是需要将参数一一传递的，这将导致效率低下。C++11 利用关键字 `using` 引入了继承构造函数的概念：

```cpp
class Base {
public:
    int value1;
    int value2;
    Base() {
        value1 = 1;
    }
    Base(int value) : Base() {
        value2 = value;
    }
};
class SubClass : public Base {
public:
    using Base::Base;
};

int main() {
    ...
    SubBase s(3);
    ...
}
```

#### 显式禁用默认函数

在传统 C++ 中，如果程序员没有提供，编译器会默认为对象生成默认构造函数、 复制构造、赋值算符以及析构函数。 另外，C++ 也为所有类定义了诸如 `new` `delete` 这样的运算符。 当程序员有需要时，可以重载这部分函数。

这就引发了一些需求：**无法精确控制默认函数的生成行为**。&#x20;

例如禁止类的拷贝时，必须将复制构造函数与赋值算符声明为 `private`。 尝试使用这些未定义的函数将导致编译或链接错误，则是一种非常不优雅的方式。

并且，编译器产生的默认构造函数与用户定义的构造函数无法同时存在。 若用户定义了任何构造函数，编译器将不再生成默认构造函数， 但有时候我们却希望同时拥有这两种构造函数，这就造成了尴尬。

C++11 提供了上述需求的解决方案，允许**显式的声明采用或拒绝编译器自带的函数**。 例如：

```cpp
class Magic {
    public:
    Magic() = default; // 显式声明使用编译器生成的构造
    Magic& operator=(const Magic&) = delete; // 显式声明拒绝编译器生成构造
    Magic(int magic_number);
}
```

#### 显式虚函数重载

在传统 C++ 中，经常容易发生意外重载虚函数的事情。例如：

```cpp
struct Base {
    virtual void foo();
}
struct SubClass: Base {
    void foo();
}
```

`SubClass::foo` 可能并不是程序员尝试重载虚函数，只是恰好加入了一个具有相同名字的函数。另一个可能的情形是，当基类的虚函数被删除后，子类拥有旧的函数就不再重载该虚拟函数并摇身一变成为了一个普通的类方法，这将造成灾难性的后果。

C++11 引入了 `override` 和 `final` 这两个关键字来防止上述情形的发生。

**override**

当重载虚函数时，引入 `override` 关键字将显式的告知编译器进行重载

编译器将检查基函数是否存在这样的虚函数，否则将无法通过编译：

```cpp
struct Base {
    virtual void foo(int);
};
struct SubClass: Base {
    virtual void foo(int) override; // 合法
    virtual void foo(float) override; // 非法, 父类没有此虚函数
};
```

**final**

`final` 则是为了防止类**被继续继承**以及**终止虚函数继续重载**引入的。

```cpp
struct Base {
    virtual void foo() final;
};

struct SubClass1 final: Base {
}; // 合法

struct SubClass2 : SubClass1 {
}; // 非法, SubClass1 已 final

struct SubClass3: Base {
    void foo(); // 非法, foo 已 final
};
```

#### 强类型枚举

传统 C++中，枚举类型并非类型安全，

枚举类型会被视作整数，则会让两种完全不同的枚举类型可以进行直接的比较（虽然编译器给出了检查，但并非所有），**甚至同一个命名空间中的不同枚举类型的枚举值名字不能相同**，这通常不是我们希望看到的结果。

C++11 引入了枚举类（enumeration class），并使用 `enum class` 的语法进行声明：

```cpp
enum class new_enum : unsigned int {
    value1,
    value2,
    value3 = 100,
    value4 = 100
};
```

这样定义的枚举实现了类型安全:

首先他不能够被隐式的转换为整数，同时也不能够将其与整数数字进行比较， 更不可能对不同的枚举类型的枚举值进行比较。

但相同枚举值之间如果指定的值相同，那么可以进行比较：

```cpp
if (new_enum::value3 == new_enum::value4) {
    ...
}
```

在这个语法中，枚举类型后面使用了**冒号及类型关键字**来指定枚举中枚举值的类型`unsigned int`，这使得我们能够为枚举赋值（未指定时将默认使用 `int`）。

而我们希望获得枚举值的值时，将必须显式的进行类型转换，不过我们可以通过重载 `<<` 这个算符来进行输出，可以收藏下面这个代码段：

```cpp
#include <iostream>
template<typename T>
std::ostream& operator<<(
    typename std::enable_if<std::is_enum<T>::value, std::ostream>::type& stream, 
    const T& e
) {
    return stream << static_cast<typename std::underlying_type<T>::type>(e);
}
```

这时，下面的代码将能够被编译：

```cpp
std::cout << new_enum::value3 << std::endl
```

### 泛型Lambda

`auto` 关键字不能够用在参数表里，这是因为这样的写法会与模板的功能产生冲突。 但是 Lambda 表达式并不是普通函数，所以在没有明确指明参数表类型的情况下，Lambda 表达式并不能够模板化。&#x20;

幸运的是，这种麻烦只存在于 C++11 中，

从 C++14 开始，Lambda 函数的形式参数可以使用 `auto` 关键字来产生意义上的泛型：

```cpp
auto add = [](auto x, auto y) {
    return x + y;
}
add(1, 2);
```

### 函数对象包装器

#### std::function

Lambda 表达式的本质是一个和**函数对象类型相似**的**类**类型（称为闭包类型）的对象（称为闭包对象）

当 Lambda 表达式的捕获列表为空时，闭包对象还能够转换为函数指针值进行传递，例如

```cpp
#include <iostream>

using foo = void(int); // 定义函数类型, using 的使用见上一节中的别名语法
void functional(foo f) { // 参数列表中定义的函数类型 foo 被视为退化后的函数指针类型 foo*
    f(1); // 通过函数指针调用函数
}

int main() {
    auto f = [](int value) {
        std::cout << value << std::endl;
    };
    functional(f); // 传递闭包对象，隐式转换为 foo* 类型的函数指针值
    f(1); // lambda 表达式调用
    return 0;
}
```

上面的代码给出了两种不同的调用形式，一种是将 Lambda 作为函数类型传递进行调用， 而另一种则是直接调用 Lambda 表达式

在 **C++11** 中，统一了这些概念，将**能够被调用的对象的类型**， 统一称之为**可调用类型**。而这种类型，便是通过 `std::function` 引入的。

C++11 `std::function` 是一种**通用、多态的函数封装**， 它的实例可以对任何可以调用的目标实体进行存储、复制和调用操作， 它也是对 C++ 中现有的可调用实体的一种类型安全的包裹（相对来说，函数指针的调用不是类型安全的）， 换句话说，就是**函数的容器**。当我们有了函数的容器之后便能够更加方便的将函数、函数指针作为对象进行处理。 例如：

```cpp
#include<functional>
#include<iostream>
int foo(int para) {
    return para;
}
int main() {
    std::function<int(int)> func = foo;
    int important = 10;
    std::function<int(int)> func2 = [&](int value) -> int {
        return 1 + value + important;
    };
    std::cout << func(10) << std::endl << func2(10) << std::endl;
}
```

#### std::bind 和 std::placeholder

`std::bind` 是用来**绑定函数调用的参数**的:

&#x20;它解决的需求是：我们有时候可能并不一定能够一次性获得调用某个函数的全部参数，通过这个函数， 我们可以将部分调用参数提前绑定到函数身上成为一个新的对象，然后在参数齐全后，完成调用。 例如：

```cpp
int foo(int a, int b, int c) {...}
int main() {
    // 将参数1,2绑定到函数 foo 上，
    // 但使用 std::placeholders::_1 来对第一个参数进行占位
    auto bindFoo = std::bind(foo, std::placeholders::_1, 1,2);
    // 这时调用 bindFoo 时，只需要提供第一个参数即可
    bindFoo(1);
}
```

**提示：**注意 `auto` 关键字的妙用。有时候我们可能不太熟悉一个函数的返回值类型， 但是我们却可以通过 `auto` 的使用来规避这一问题的出现。

### 右值

C++11 中为了引入强大的右值引用，将右值的概念进行了进一步的划分，分为：纯右值、将亡值。

**纯右值** (prvalue, pure rvalue)，纯粹的右值，要么是纯粹的字面量，例如 `10`, `true`； 要么是求值结果相当于字面量或匿名临时对象，例如 `1+2`。非引用返回的临时变量、运算表达式产生的临时变量、 原始字面量、Lambda 表达式都属于纯右值。

需要注意的是，字面量除了字符串字面量以外，均为纯右值。而字符串字面量是一个左值，类型为 `const char` 数组。例如：

```cpp
#include <type_traits>

int main() {
    // 正确，"01234" 类型为 const char [6]，因此是左值
    const char (&left)[6] = "01234";

    // 断言正确，确实是 const char [6] 类型，注意 decltype(expr) 在 expr 是左值
    // 且非无括号包裹的 id 表达式与类成员表达式时，会返回左值引用
    static_assert(std::is_same<decltype("01234"), const char(&)[6]>::value, "");

    // 错误，"01234" 是左值，不可被右值引用
    // const char (&&right)[6] = "01234";
}
```

但是注意，数组可以被隐式转换成相对应的指针类型，而转换表达式的结果（如果不是左值引用）则一定是个右值（右值引用为将亡值，否则为纯右值）。例如：

```cpp
const char*   p   = "01234";  // 正确，"01234" 被隐式转换为 const char*
const char*&& pr  = "01234";  // 正确，"01234" 被隐式转换为 const char*，该转换的结果是纯右值
// const char*& pl = "01234"; // 错误，此处不存在 const char* 类型的左值
```

**将亡值**(xvalue, expiring value)，是 C++11 为了引入右值引用而提出的概念（因此在传统 C++ 中， 纯右值和右值是同一个概念），也就是即将被销毁、却能够被移动的值。

将亡值可能稍有些难以理解，我们来看这样的代码：

```cpp
std::vector<int> foo() {
    std::vector<int> temp = {1, 2, 3, 4};
    return temp;
}

std::vector<int> v = foo();
```

在这样的代码中，就传统的理解而言，函数 `foo` 的返回值 `temp` 在内部创建然后被赋值给 `v`， 然而 `v` 获得这个对象时，会将整个 `temp` 拷贝一份，然后把 `temp` 销毁，如果这个 `temp` 非常大， 这将造成大量额外的开销（这也就是传统 C++ 一直被诟病的问题）。

在最后一行中，`v` 是左值、 `foo()` 返回的值就是右值（也是纯右值）。

但是，`v` 可以被别的变量捕获到， 而 `foo()` 产生的那个返回值作为一个临时值，一旦被 `v` 复制后，将立即被销毁，无法获取、也不能修改。&#x20;

而将亡值就定义了这样一种行为：**临时的值、能够被识别、同时又能够被移动**。

在 C++11 之后，编译器为我们做了一些工作，此处的左值 `temp` 会被进行此隐式右值转换， 等价于 `static_cast<std::vector<int> &&>(temp)`，进而此处的 `v` 会将 `foo` 局部返回的值进行移动。 也就是后面我们将会提到的移动语义。

要拿到一个将亡值，就需要用到**右值引用：`T &&`**。&#x20;

右值引用的声明让这个临时值的生命周期得以延长、只要变量还活着，那么将亡值将继续存活。

C++11 提供了 `std::move` 这个方法将**左值参数无条件的转换为右值**， 有了它我们就能够方便的获得一个右值临时对象，例如：

```cpp
void reference(std::string& str) {
    std::cout << "left" << std::end;
}
void reference(std::string&& str) {
    std::cout << "右值" << std::endl;
}
int main() {
    std::string lv1 = "string,"; // lv1 是一个左值
    // std::string&& r1 = lv1; // 非法, 右值引用不能引用左值
    std::string&& rv1 = std::move(lv1); // 合法, std::move可以将左值转移为右值
    std::cout << rv1 << std::endl; // string,

    const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
    // lv2 += "Test"; // 非法, 常量引用无法被修改
    std::cout << lv2 << std::endl; // string,string,

    std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
    rv2 += "Test"; // 合法, 非常量引用能够修改临时变量
    std::cout << rv2 << std::endl; // string,string,string,Test

    reference(rv2); // 输出左值

    return 0;
} 
```

传统 C++ 通过拷贝构造函数和赋值操作符为类对象设计了拷贝/复制的概念，但为了实现对资源的移动操作， 调用者必须使用先复制、再析构的方式，否则就需要自己实现移动对象的接口。 试想，搬家的时候是把家里的东西直接搬到新家去，而不是将所有东西复制一份（重买）再放到新家、 再把原来的东西全部扔掉（销毁），这是非常反人类的一件事情。

传统的 C++ 没有区分『**移动**』和『**拷贝**』的概念，造成了大量的数据拷贝，浪费时间和空间。 右值引用的出现恰好就解决了这两个概念的混淆问题，例如：

```cpp
#include <iostream>
class A {
public:
    int *pointer;
    A():pointer(new int(1)) {
        std::cout << "构造" << pointer << std::endl;
    }
    A(A& a):pointer(new int(*a.pointer)) {
        std::cout << "拷贝" << pointer << std::endl;
    } // 无意义的对象拷贝
    A(A&& a):pointer(a.pointer) {
        a.pointer = nullptr;
        std::cout << "移动" << pointer << std::endl;
    }
    ~A(){
        std::cout << "析构" << pointer << std::endl;
        delete pointer;
    }
};
// 防止编译器优化
A return_rvalue(bool test) {
    A a, b;
    if(test) return a; // 等价于 static_cast<A&&>(a);
    else return b;     // 等价于 static_cast<A&&>(b);
}
int main() {
    A obj = return_rvalue(false);
    std::cout << "obj:" << std::endl;
    std::cout << obj.pointer << std::endl;
    std::cout << *obj.pointer << std::endl;
    return 0;
}
```

在上面的代码中：

1. 首先会在 `return_rvalue` 内部构造两个 `A` 对象，于是获得两个构造函数的输出；
2. 函数返回后，产生一个将亡值，被 `A` 的移动构造（`A(A&&)`）引用，从而延长生命周期，并将这个右值中的指针拿到，保存到了 `obj` 中，而将亡值的指针被设置为 `nullptr`，防止了这块内存区域被销毁。

从而避免了无意义的拷贝构造，加强了性能。再来看看涉及标准库的例子：

```cpp
#include <iostream> // std::cout
#include <utility> // std::move
#include <vector> // std::vector
#include <string> // std::string

int main() {

    std::string str = "Hello world.";
    std::vector<std::string> v;

    // 将使用 push_back(const T&), 即产生拷贝行为
    v.push_back(str);
    // 将输出 "str: Hello world."
    std::cout << "str: " << str << std::endl;

    // 将使用 push_back(const T&&), 不会出现拷贝行为
    // 而整个字符串会被移动到 vector 中，所以有时候 std::move 会用来减少拷贝出现的开销
    // 这步操作后, str 中的值会变为空
    v.push_back(std::move(str));
    // 将输出 "str: "
    std::cout << "str: " << str << std::endl;

    return 0;
}
```

#### 完美转发

前面我们提到了，一个声明的右值引用其实是一个左值。这就为我们进行参数转发（传递）造成了问题：

```cpp
void reference(int& v) {
    std::cout << "左值" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "普通传参:";
    reference(v); // 始终调用 reference(int&)
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1); // 1是右值, 但输出是左值

    std::cout << "传递左值:" << std::endl;
    int l = 1;
    pass(l); // l 是左值, 输出左值

    return 0;
}

```

对于 `pass(1)` 来说，虽然传递的是右值，但由于 `v` 是一个引用，所以同时也是左值。 因此 `reference(v)` 会调用 `reference(int&)`，输出『左值』。 而对于`pass(l)`而言，`l`是一个左值，为什么会成功传递给 `pass(T&&)` 呢？

这是基于**引用坍缩规则**的：在传统 C++ 中，我们不能够对一个引用类型继续进行引用， 但 C++ 由于右值引用的出现而放宽了这一做法，从而产生了引用坍缩规则，允许我们对引用进行引用， 既能左引用，又能右引用。但是却遵循如下规则：

| 函数形参类型 | 实参参数类型 | 推导后函数形参类型 |
| :----: | :----: | :-------: |
|   T&   |   左引用  |     T&    |
|   T&   |   右引用  |     T&    |
|   T&&  |   左引用  |     T&    |
|   T&&  |   右引用  |    T&&    |

因此，模板函数中使用 `T&&` 不一定能进行右值引用，当传入左值时，此函数的引用将被推导为左值。 更准确的讲，**无论模板参数是什么类型的引用，当且仅当实参类型为右引用时，模板参数才能被推导为右引用类型**。 这才使得 `v` 作为左值的成功传递。

完美转发就是基于上述规律产生的。所谓完美转发，就是为了让我们在传递参数的时候， 保持原来的参数类型（左引用保持左引用，右引用保持右引用）。 为了解决这个问题，我们应该使用 `std::forward` 来进行参数的转发（传递）：

```cpp
#include <iostream>
#include <utility>
void reference(int& v) {
    std::cout << "左值引用" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值引用" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "              普通传参: ";
    reference(v);
    std::cout << "       std::move 传参: ";
    reference(std::move(v));
    std::cout << "    std::forward 传参: ";
    reference(std::forward<T>(v));
    std::cout << "static_cast<T&&> 传参: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1);

    std::cout << "传递左值:" << std::endl;
    int v = 1;
    pass(v);

    return 0;
}
```

输出结果为：

```cpp
传递右值:
              普通传参: 左值引用
       std::move 传参: 右值引用
    std::forward 传参: 右值引用
static_cast<T&&> 传参: 右值引用
传递左值:
              普通传参: 左值引用
       std::move 传参: 右值引用
    std::forward 传参: 左值引用
static_cast<T&&> 传参: 左值引用

```

无论传递参数为左值还是右值，普通传参都会将参数作为左值进行转发， 所以 `std::move` 总会接受到一个左值，从而转发调用了`reference(int&&)` 输出右值引用。

唯独 `std::forward` 即没有造成任何多余的拷贝，同时**完美转发**(传递)了函数的实参给了内部调用的其他函数。

`std::forward` 和 `std::move` 一样，没有做任何事情，`std::move` 单纯的将左值转化为右值， `std::forward` 也只是单纯的将参数做了一个类型的转换，从现象上来看， `std::forward<T>(v)` 和 `static_cast<T&&>(v)` 是完全一样的。

读者可能会好奇，为何一条语句能够针对两种类型的返回对应的值， 我们再简单看一看 `std::forward` 的具体实现机制，`std::forward` 包含两个重载：

```cpp
template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type& __t) noexcept
{ return static_cast<_Tp&&>(__t); }

template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
{
    static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
        " substituting _Tp is an lvalue reference type");
    return static_cast<_Tp&&>(__t);
}
```

在这份实现中，`std::remove_reference` 的功能是消除类型中的引用， `std::is_lvalue_reference` 则用于检查类型推导是否正确，在 `std::forward` 的第二个实现中 检查了接收到的值确实是一个左值，进而体现了坍缩规则。

当 `std::forward` 接受左值时，`_Tp` 被推导为左值，所以返回值为左值；而当其接受右值时， `_Tp` 被推导为 右值引用，则基于坍缩规则，返回值便成为了 `&& + &&` 的右值。 可见 `std::forward` 的原理在于巧妙的利用了模板类型推导中产生的差异。

这时我们能回答这样一个问题：为什么在使用循环语句的过程中，`auto&&` 是最安全的方式？ 因为当 `auto` 被推导为不同的左右引用时，与 `&&` 的坍缩组合是完美转发。

### 元组

纵观传统 C++ 中的容器，除了 `std::pair` 外， 似乎没有现成的结构能够用来存放不同类型的数据（通常我们会自己定义结构）。 但 `std::pair` 的缺陷是显而易见的，只能保存两个元素。

#### 基本操作

关于元组的使用有三个核心的函数：

1. `std::make_tuple`: 构造元组
2. `std::get`: 获得元组某个位置的值
3. `std::tie`: 元组拆包

```cpp
#include<tuple>
#include<iostream>

auto get_student(int id) {
    // 返回类型被推断为 std::tuple<double, char, std::string>
    if (id == 0) {
        return std::make_tuple(3.8, 'A', "zhangsan");
    } else if (id == 1) {
        return std::make_tuple(2.9, 'C', "李四");
    } else if (id == 2) {
        return std::make_tuple(1.7, 'D', "王五");
    }
    return std::make_tuple(0.0, 'D', "null");
    // 如果只写 0 会出现推断错误, 编译失败!!!!!
}

int main() {
    auto student = get_student(0);
    std:cout << "ID: 0, "
    << "GPA: " << std::get<0>(student) << ", "
    << "成绩: " << std::get<1>(student) << ", "
    << "姓名: " << std::get<2>(student) << '\n';

    double gpa;
    char grade;
    std::string name;

    // 元组进行拆包
    std::tie(gpa, grade, name) = get_student(1);
    std::cout << "ID: 1, "
    << "GPA: " << gpa << ", "
    << "成绩: " << grade << ", "
    << "姓名: " << name << '\n';
}   
```

`std::get` 除了使用常量获取元组对象外，**C++14** 增加了使用类型来获取元组中的对象：

```cpp
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
std::cout << std::get<std::string>(t) << std::endl;
std::cout << std::get<double>(t) << std::endl; // 非法, 引发编译期错误
std::cout << std::get<3>(t) << std::endl;
```

#### 运行期索引

如果你仔细思考一下可能就会发现上面代码的问题，`std::get<>` 依赖一个编译期的常量，所以下面的方式是不合法的：

```cpp
int index = 1;
std::get<index>(t);
```

那么要怎么处理？

答案是，使用 `std::variant<>`（**C++ 17** 引入），提供给 `variant<>` 的类型模板参数

可以让一个 `variant<>` 从而容纳提供的几种类型的变量（在其他语言，例如 Python/JavaScript 等，表现为动态类型）：

```cpp
#include<variant>
template<size_t n, typename... T>
constexpr std::variant<T...> _tuple_index(const std::tuple<T...>& tp1, size_t i) {
    if constexpr (n >= sizeof...(T)) {
        throw std::out_of_range("越界");
    }
    if (i == n) {
        return std::variant<T...> {
            std::in_place_index<n>,
            std::get<n>(tp1)
        };
    }
    return _tuple_index<(n < sizeof...(T) - 1 ? n + 1 : 0)>(tp1,i);
}
constexpr std::variant<T...> tuple_index(const std::tuple<T...>& tpl, size_t i) {
    return _tuple_index<0>(tpl, i);
}
template <typename T0, typename... Ts>
std::ostream & operator<< (std::ostream & s, std::variant<T0, Ts...> const & v) {
    std::visit([&](auto && x) {
        s << x;
    }, v);
    return s;
}
```

这样我们就能：

```cpp
int index = 1;
std::cout << tuple_index(t, index) << std::endl;
```

#### 元组合并和遍历

还有一个常见的需求就是合并两个元组，这可以通过 `std::tuple_cat` 来实现：

```cpp
auto new_tuple = std::tuple_cat(get_student(1), std::move(t));
```

马上就能够发现，应该如何快速遍历一个元组？但是我们刚才介绍了如何在运行期通过非常数索引一个 `tuple` 那么遍历就变得简单了， 首先我们需要知道一个元组的长度，可以：

```cpp
template <typename T>
auto tuple_len(T &tp1) {
    return std::tuple_size<T>::value;
}
```

这样就能够对元组进行迭代了：

```cpp
for (int i = 0; i != tuple_len(new_tuple); ++i) {
    std::cout << tuple_index(new_tuple, i) << std::endl;
}
```

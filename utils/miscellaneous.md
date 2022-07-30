---
description: 一些C++11/14/17小点
---

# 😅 Miscellaneous

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

#### 显式
















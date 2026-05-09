# Modern C++ Language Basics

---
description: Selected C++11/14/17 language notes
---

# Miscellaneous

> Start with the most comprehensive learning resources:
>
> ****[**My favorite C++ talk channel**](https://www.youtube.com/user/CppCon/videos)****
>
> Official documentation:
>
> [https://en.cppreference.com/w/cpp/compiler\_support](https://en.cppreference.com/w/cpp/compiler\_support)
>
> [https://en.cppreference.com/w](https://en.cppreference.com/w)

### `long long int`

`long long int` was not first introduced by C++11. It had already been added to the C standard in C99, so most compilers supported it before C++11.&#x20;

C++11 formally incorporated it into the C++ standard and requires `long long int` to provide at least 64 bits.

### `noexcept` as a Specifier and Operator

One major advantage of C++ over C is that C++ defines a full exception-handling mechanism. Before C++11, however, dynamic exception specifications written after function declarations were rarely used. Starting in C++11, that mechanism was deprecated, so this note does not discuss the old form or how it worked; it is not something worth learning proactively for modern C++.

C++11 simplifies exception declarations into two practical cases:

1. A function may throw exceptions.
2. A function must not throw exceptions.

`noexcept` is used to mark the second behavior. For example:

`void may_throw(); // may throw exceptions`&#x20;

`void no_throw() noexcept; // must not throw exceptions`

If a function declared `noexcept` throws an exception, the runtime calls `std::terminate()` and immediately terminates the program.

`noexcept` can also be used as an operator on an expression. It returns `true` when the expression is known not to throw, and `false` otherwise.

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

After a function is marked `noexcept`, it becomes an exception boundary: if an exception escapes from inside it, callers do not catch that exception because the program terminates instead. For example:

```cpp
try {  
    may_throw();  
} catch (...) {  
    std::cout << "caught exception from may_throw()" << std::endl;
}  
try {  
    non_block_throw();  
} catch (...) {  
    std::cout << "caught exception from non_block_throw()" << std::endl;
}  
try {  
    block_throw();  // the function is declared noexcept
} catch (...) {  
    std::cout << "caught exception from block_throw()" << std::endl;
}  
```

The final output is:

```
caught exception from may_throw()
caught exception from non_block_throw()
```

### Literals

#### Raw String Literals

In traditional C++, writing strings that contain many special characters is painful. For example, a string containing HTML often needs many escape characters, and a Windows file path is usually written as `C:\\File\\To\\Path`.

C++11 provides raw string literals. You can prefix a string with <mark style="color:red;">**`R`**</mark> and wrap the raw content in parentheses:

```cpp
#include <iostream>  
#include <string>  
int main() {  
    std::string str = R"(C:\File\To\Path)";  
    std::cout << str << std::endl;  
    return 0;  
}  
```

#### User-Defined Literals

C++11 introduced user-defined literals, implemented by overloading literal suffix operators:

Custom string literals must use the following parameter list:

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

User-defined literals support four literal categories:

1. **Integer literals**: overloads must use `unsigned long long`, `const char *`, or a literal operator template parameter. The example above uses the first form.
2. **Floating-point literals**: overloads must use `long double`, `const char *`, or a literal operator template.
3. **String literals**: overloads must use the `(const char *, size_t)` parameter form.
4. **Character literals**: the parameter type can only be `char`, `wchar_t`, `char16_t`, or `char32_t`.

### Memory Alignment

C++11 introduced two keywords, **`alignof`** and **`alignas`**, for controlling memory alignment. `alignof` returns a platform-dependent `std::size_t` value that lets us **query** a type's alignment requirement. Sometimes querying is not enough and we want to define the alignment of a structure ourselves. For that, C++11 introduced `alignas`, which can **specify** the alignment requirement of a type or object. Consider two examples:

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

`std::max_align_t` is a type whose alignment requirement is at least as strict as that of every scalar type. On many platforms this corresponds to the alignment of `long double`, so `AlignasStorage` commonly ends up requiring 8-byte or 16-byte alignment.

### C++11 Deprecations and Replacements

*   Assigning a string literal to `char *` is no longer allowed in modern C++. If you need to initialize from a string literal, use `const char *` or `auto`.

    For example:

    ```
    char *str = "hello world!";  // emits a deprecation warning or error
    ```
* C++98 dynamic exception specifications, `unexpected_handler`, `set_unexpected()`, and related features were deprecated. Use `noexcept`.
* `auto_ptr` was deprecated.
* The `register` keyword was deprecated. It may still be parsed in older language modes, but it no longer has practical meaning.
* The `++` operation on `bool` was deprecated and later removed.
* Implicit generation of a copy constructor and copy-assignment operator for a class that declares a destructor was deprecated.
* C-style casts, written as `(convert_type)` before an expression, should be avoided in favor of `static_cast`, `reinterpret_cast`, and `const_cast`.
* C++17 removed or deprecated some C compatibility headers such as `<ccomplex>`, `<cstdalign>`, `<cstdbool>`, and `<ctgmath>`.

### Constants

#### nullptr

`nullptr` was introduced to replace `NULL`. In traditional C++, `NULL` and `0` are often treated as the same thing, depending on how the compiler or standard library defines `NULL`. Some C environments define `NULL` as `((void*)0)`, while C++ implementations commonly define it as `0` or `0L`.

C++ **does not allow** an implicit conversion from `void *` to arbitrary pointer types. If `NULL` were defined as `((void*)0)`, code such as `char *ch = NULL;` would fail under C++'s conversion rules. Therefore C++ implementations typically define `NULL` as an integer null pointer constant. That creates another problem: defining `NULL` as `0` can interact badly with overload resolution.

For example, if both `void f(char*)` and `void f(int)` exist, `f(NULL)` may call the integer overload.

C++11 introduced the `nullptr` keyword to distinguish null pointers from the integer value `0`. The type of `nullptr` is `std::nullptr_t`. It can be implicitly converted to any pointer or pointer-to-member type, and it can be compared for equality or inequality with such pointer values.

#### constexpr

If the compiler can evaluate constant expressions such as `1 + 2` at compile time and embed the result into the generated program, runtime performance can improve.

Here is a small example:

```cpp
#include <iostream>
#define LEN 10 // constant-like macro

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
    char arr_1[10];                      // valid
    char arr_2[LEN];                     // valid

    int len = 10;
    // char arr_3[len];                  // invalid

    const int len_2 = len + 1;
    constexpr int len_2_constexpr = 1 + 2 + 3;
    // char arr_4[len_2];                // invalid
    char arr_4[len_2_constexpr];         // valid

    // char arr_5[len_foo()+5];          // invalid
    char arr_6[len_foo_constexpr() + 1]; // valid

    std::cout << fibonacci(10) << std::endl;
    // 1, 1, 2, 3, 5, 8, 13, 21, 34, 55
    std::cout << fibonacci(10) << std::endl;
    return 0;
}
```

In the example above, `char arr_4[len_2]` may look confusing because `len_2` has already been declared as a constant.

**Why is `char arr_4[len_2]` still invalid?**

The reason is that **the C++ standard requires an array bound to be a constant expression**. `len_2` is a `const` object, but it is not a constant expression because its initializer depends on the runtime variable `len`. Even if many compilers accept similar code as an extension, it is not standard C++. C++11's `constexpr` addresses this distinction.

**Notes on constants and constant expressions:**

A constant is a fixed value that does not change during program execution. A **constant expression** is an expression that can be evaluated at **compile time**. Constant expressions can be used as non-type template arguments, array bounds, and in other contexts that require compile-time values.

A constant is not automatically a constant expression. Only a constant initialized by a constant expression can itself be used as a constant expression. A constant initialized from a non-constant expression, such as `len_2` above, is merely a constant object. If a constant's initializer is not a constant expression, the constant is not usable as a constant expression.

* To define a constant variable, place `const` before or after the variable type.
* Defining a `const` variable without initializing it causes a compilation error.
* A constant variable may be initialized from an ordinary variable.
*   C++ has two broad categories of constants: compile-time constants and runtime constants.

    Runtime constants are initialized only when the program runs. Compile-time constants are initialized during compilation.

    In many contexts, compile-time constants and runtime constants can be used in similar ways.

    In special contexts, however, C++ requires compile-time constants. C++11 introduced `constexpr` to make that distinction explicit.

A related practice is to define symbolic constants:

In many programs, a symbolic constant is used throughout the codebase, such as a mathematical or physical constant. Instead of redefining it repeatedly, define it once in a central place and reuse it elsewhere. If the value needs to change, only one location needs to be updated.

There are many ways to do this in C++, but the following is one of the simplest:

* Create a header file that contains the constants.
* Define a namespace in that header.
* Put all related constants in the namespace.
* Include the header wherever the constants are needed.

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

Use the scope-resolution operator `::` to reference the constants:

```objectivec
#include "constants.h"
double circumference = 2 * radius * constants::pi;
```

If you have both physical constants and application tuning values, split them into two files: one for physical constants that should never change, and another for tuning values that may change. The physical constants can then be reused broadly without being mixed with application policy.

Returning to the earlier function example:

For `arr_5`, pre-C++11 compilers had no way to know that `len_foo()` effectively returns a constant value at runtime, which makes the array bound invalid as a standard constant expression.

C++11 provides `constexpr`, allowing users to **explicitly declare that a function or object constructor can produce a constant expression**. The keyword tells the compiler to verify that the function can be evaluated at compile time when used in a constant-expression context.

In addition, `constexpr` functions can use recursion.

Starting in C++**14**, `constexpr` functions can **use simple statements such as local variables, loops, and branches** internally. The following code, for example, does **not** compile as a C++**11** `constexpr` function:

```cpp
constexpr int fibonacci(const int n) {
    if(n == 1) return 1;
    if(n == 2) return 1;
    return fibonacci(n-1) + fibonacci(n-2);
}
```

For C++11, we therefore write the simplified expression-form version shown earlier.

### Variables

In traditional C++, variable declarations can appear in many locations, and a temporary `int` can even be declared inside a `for` statement. However, there was no way to declare an initializer variable directly inside an `if` or `switch` statement. For example:

```cpp
int main() {
    std::vector<int> vec = {1, 2, 3, 4};

    // before C++17
    const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 2);
    if (itr != vec.end()) {
        *itr = 3;
    }

    // need to define a new variable
    const std::vector<int>::iterator itr2 = std::find(vec.begin(), vec.end(), 3);
    if (itr2 != vec.end()) {
        *itr2 = 4;
    }

    // prints 1, 4, 3, 4
    for (std::vector<int>::iterator element = vec.begin(); element != vec.end(); 
        ++element)
        std::cout << *element << std::endl;
}
```

In the code above, `itr` is defined in the whole `main()` scope. When we need to search the same `std::vector` again, we must introduce another variable name.

**C++17 removes this limitation by allowing an initializer inside `if` and `switch`:**

```cpp
// place the temporary variable inside the if statement
if (const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 3);
    itr != vec.end()) {
    *itr = 4;
}
```

#### Initializer Lists

The most common use case is object initialization.

In traditional C++, different objects use different initialization forms. Ordinary arrays and POD (**P**lain **O**ld **D**ata, meaning classes or structs without constructors, destructors, or virtual functions) can be initialized with `{}`, which is what we call an initializer list. Class objects, however, are initialized either through copy construction or with `()`. These forms apply to different categories and are not unified.

To solve this, C++11 first binds the concept of an initializer list to a type, <mark style="color:green;background-color:green;">`std::initializer_list`</mark>. Constructors and ordinary functions can then accept initializer lists as parameters, creating a bridge between **class-object initialization** and **ordinary array/POD initialization**. For example:

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
    // since C++11
    MagicFoo magicFoo = {1, 2, 3, 4, 5};

    std::cout << "magicFoo: ";
    for (std::vector<int>::iterator it = magicFoo.vec.begin(); 
        it != magicFoo.vec.end(); ++it) {
        std::cout << *it << std::endl;
    }
}
```

This kind of constructor is called an initializer-list constructor. Types that provide such a constructor receive special treatment during initialization.

Initializer lists can be used not only for object construction, but also as ordinary function parameters:

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

C++11 also provides a unified syntax for initializing arbitrary objects:

```
Foo foo2 {3, 4};
```

#### Structured Bindings

Structured bindings provide a feature similar to multiple return values in other languages. C++11 added `std::tuple`, which can package multiple return values. The limitation is that C++11/14 did not provide a simple way to directly extract and define the elements of a tuple. Although `std::tie` can unpack a tuple, the programmer still needs to know exactly how many objects the tuple contains and what their types are.

C++**17** improves this with structured bindings, allowing code like this:

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

### Type Deduction

C++11 introduced `auto` and `decltype` for type deduction, allowing the compiler to infer variable types.

This gives C++ a programming style closer to other modern languages in which programmers do not always need to spell out variable types manually.

#### auto

`auto` has existed in C++ for a long time, but it originally served as a storage-class specifier alongside `register`. In traditional C++, if a variable was not declared as `register`, it was automatically treated as an `auto` variable. As `register` became deprecated and, in C++17, reserved for future use with no practical meaning, changing the meaning of `auto` became natural.

One of the most common and visible uses of `auto` is iterator type deduction. Earlier sections showed the verbose traditional C++ iterator style:

```cpp
// before C++11
// because cbegin() returns vector<int>::const_iterator,
// itr should also have type vector<int>::const_iterator
for(vector<int>::const_iterator it = vec.cbegin(); itr != vec.cend(); ++it)
```

With `auto`, this can be written more directly:

```cpp
class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
        // since C++11, use auto for type deduction
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

Some other common usages:

```cpp
auto i = 5;              // i is deduced as int
auto arr = new auto(10); // arr is deduced as int *
```

Starting in **C++20**, `auto` can even be used in function parameters. Consider this example:

```cpp
int add(auto x, auto y) {
    return x+y;
}

auto i = 5; // deduced as int
auto j = 6; // deduced as int
std::cout << add(i, j) << std::endl;

```

**Note**: `auto` still cannot be used to deduce an array type in this form:

```cpp
auto auto_arr2[10] = {arr}; // error: cannot deduce array element type
```

#### decltype(auto)

`decltype(auto)` is a slightly more advanced feature introduced in C++14.

In short, `decltype(auto)` is mainly used to deduce the return type of forwarding functions or wrapper functions. It avoids explicitly writing the expression parameter to `decltype`. Consider this example, where we need to wrap the following two functions:

```cpp
std::string  lookup1();
std::string& lookup2();
```

In C++11, the wrappers would look like this:

```cpp
std::string look_up_a_string_1() {
    return lookup1();
}
std::string& look_up_a_string_2() {
    return lookup2();
}
```

With `decltype(auto)`, the compiler can handle this tedious return-type forwarding:

```cpp
decltype(auto) look_up_a_string_1() {
    return lookup1();
}
decltype(auto) look_up_a_string_2() {
    return lookup2();
}
```

### Control Flow

#### if constexpr

&#x20;C++11 introduced `constexpr`, which allows expressions or functions to produce compile-time constant results. A natural extension is to bring that idea into conditional branching: if a branch can be selected at compile time, the generated program can avoid irrelevant runtime code.

**C++17** introduced `constexpr` into `if` statements, **allowing a branch condition to be evaluated as a constant expression**. Consider this code:

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

// at compile time, the effective generated code behaves like this:
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

#### Range-Based `for`

**C++11** introduced range-based iteration syntax:

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

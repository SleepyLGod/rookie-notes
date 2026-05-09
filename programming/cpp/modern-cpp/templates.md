# Modern C++ Templates

### template

#### Extern Templates

In traditional C++, templates are instantiated by the compiler only when they are used.

In other words, whenever compiled code in a translation unit sees a fully defined template and needs it, the compiler may instantiate it. This can produce repeated instantiations across translation units and increase compilation time. Before C++11, there was no direct way to tell the compiler not to instantiate a template in a given translation unit.

For this reason, **C++11** introduced extern templates. This extends the original syntax for forcing template instantiation at a specific location and lets us explicitly tell the compiler **where template instantiation should or should not happen**:

```cpp
template class std::vector<bool>; // force instantiation
extern template class std::vector<double>; // do not instantiate in this translation unit
```

#### Closing Angle Brackets `>`

In traditional C++ compilers, `>>` was always parsed as the right-shift operator. This caused a problem for nested templates: code such as `std::vector<std::vector<int>> matrix;` did not compile in traditional C++ and had to be written with a separating space.

Starting in C++11, consecutive closing angle brackets became legal and compile correctly. Even code like the following can compile:

```cpp
template<bool T>
class MType {
    bool m = T;
}
int main() {
 ...
 std::vector<MType(1>2)>> m; // valid, but do not write code like this
```

#### Alias Templates

Consider this statement carefully: **templates are used to generate types.**

In traditional C++, `typedef` can define a new name for a type, but it cannot define a new name for a template because a template itself is not a type. For example:

```cpp
template<typename T, typename U>
class MagicType {
public:
    T dark;
    U magic;
};

// invalid
template<typename T>
typedef MagicType<std::vector<T>, std::string> FakeDarkMagic;
```

**C++11** introduced `using` aliases, which support both alias templates and the ordinary use cases previously handled by `typedef`:

Normally, a `typedef` alias is written as `typedef original_name new_name;`. However, function-pointer aliases and similar cases use a different-looking syntax, which often makes direct reading harder.

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

#### Variadic Templates

Before C++11, both class templates and function templates accepted only a fixed number of template parameters specified by their declarations.

C++11 added new syntax that allows an arbitrary number of template parameters of arbitrary types, without fixing the number of parameters at definition time.

```cpp
template<typename... Ts> class Magic;
```

An object of the template class `Magic` can accept an unrestricted number of `typename` template parameters. For example:

```cpp
Magic<int,
      std::vector<int>,
      std::map<std::string, std::vector<int>>> darkMagic;
```

Because the parameter pack can have arbitrary length, a template argument count of `0` is also allowed: `Magic<> nothing;`.

If a zero-length parameter pack should not be allowed, define at least one required template parameter explicitly:

```cpp
template<typename Require, typename... Args> class Magic2;
```

Variadic templates can also be applied directly to function templates.

The traditional C `printf` function supports calls with a variable number of arguments, but it is not type-safe. C++11 not only lets us define type-safe variadic functions, but also lets `printf`-like functions naturally handle user-defined object types. Besides using `...` in the template parameter list to represent a template parameter pack, **function parameters** use the same notation to represent a function parameter pack. This makes it convenient to write variadic functions:

```cpp
template<typename... Args> 
void printf(const std::string &str, Args... args);
```

After defining variadic template parameters, how do we unpack them?

First, we can use `sizeof...` to count the number of parameters:

```cpp
template<typename... Args>
void magic(Args... args) {
    std::cout << sizeof...(args) << std::endl;
}
```

We can pass any number of arguments to the `magic` function.

Second, we need to unpack the parameters. There is no single pre-C++17 syntax that magically handles every parameter pack, but there are several classic techniques:

**1. Recursive Template Functions**

Recursion is the most obvious and classic approach. This method repeatedly **passes the remaining template parameters to another function call**, recursively traversing all template arguments:

```cpp
template<typename T0>
void printf0(T0 value) { // single argument: final recursion step
    std::cout << value << std::endl;
}
template<typename T, typename... Ts>
void printf0(T value, Ts... args) { // multiple arguments: recursive case
    std::cout << value << std::endl;
    printf0(args...);
}
int main() {
    printf0(1, 2, "123", 1.1);
    return 0;
}
```

**2. Variadic Template Expansion**

The recursive style is verbose. C++17 added better support for variadic expansion patterns, so the `printf` example can be written in a single function:

```cpp
template<typename T0, typename... Ts>
void printf1(T0 t0, Ts... t) {
    std::cout << t0 << std::endl;
    if constexpr (sizeof...(t) > 0) {
        printf1(t...);
    }
}
```

In practice, even when we use variadic templates, we do not always need to traverse the parameters one by one.

We can also use features such as **`std::bind`** and perfect forwarding to bind functions and arguments and then perform the call.

**3. Initializer-List Expansion**

Recursive template functions are a standard technique, but their obvious drawback is that a separate recursion-termination function must be defined.

Here is a trick that uses initializer-list expansion:

```cpp
template<typename T, typename... Ts>
auto printf2(T value, Ts... args) {
    std::cout << value << std::endl;
    (void) std::initializer_list<T> { ( [&args] {
        std::cout << args << std::endl;
    }(), value)...};
}
```

This code additionally uses initializer lists and lambda expressions introduced in C++11. Lambda expressions are covered in the next section.

Through the initializer list, `(lambda expression, value)...` is expanded.

Because the comma expression is used, the lambda expression on the left executes first and prints the argument.&#x20;

To avoid compiler warnings, we explicitly cast the `std::initializer_list` expression to `void`.

#### Fold Expressions

**C++17** extends variadic parameter support to expressions through fold expressions. Consider this example:

```cpp
template<typename ... T>
auto sum(T ... t) {
    return (t + ...);
}
int main() {
    std::cout << sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) << std::endl;
}
```

#### Non-Type Template Parameter Deduction

So far, the main kind of template parameter discussed has been the type template parameter.

```cpp
template <typename T, typename U>
auto add(T t, U u) {
    return t+u;
}
```

Here, the template parameters `T` and `U` represent concrete types.&#x20;

There is another common form of template parameter that allows **literal values to become template parameters**: non-type template parameters.

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

With this template-parameter form, we can pass `100` as a template argument.&#x20;

After **C++11** introduced type deduction, a natural question arises: since this template argument is passed as a concrete literal, can the compiler deduce its type for us through the placeholder `auto`, so that we no longer need to explicitly specify the type?&#x20;

Fortunately, **C++17** introduced this feature. We can use the `auto` keyword and let the compiler deduce the concrete type:

```cpp
template <auto value> 
void foo() {
    std::cout << value << std::endl;
    return;
}

int main() {
    foo<10>();  // value is deduced as int
}
```

### Object-Oriented Features

#### Delegating Constructors

C++11 introduced delegating constructors, allowing **one constructor to call another constructor in the same class**, which simplifies code:

```cpp
class Base {
public:
    int value1;
    int value2;
    Base() {
        value1 = 1;
    }
    Base(int value) : Base() { // delegate to the Base() constructor
        value2 = value;
    }
};
```

#### Inheriting Constructors

In traditional C++, if derived classes needed constructor behavior from a base class, they often had to forward constructor parameters manually, which was repetitive and inefficient. C++11 introduced inheriting constructors through the `using` keyword:

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

#### Explicitly Defaulted and Deleted Functions

In traditional C++, if the programmer does not provide them, the compiler may generate a default constructor, copy constructor, assignment operator, and destructor for a class. C++ also defines operators such as `new` and `delete` for all classes, and programmers can overload them when needed.

This creates a need that older C++ did not handle cleanly: **precise control over generation of default functions**.&#x20;

For example, to make a class non-copyable, older code commonly declared the copy constructor and assignment operator as `private` and left them undefined. Attempts to use those functions would then cause compilation or linking errors. This is an inelegant technique.

Also, an implicitly generated default constructor and user-defined constructors do not always coexist. If the user defines any constructor, the compiler no longer implicitly generates a default constructor. Sometimes we want both, which creates friction.

C++11 solves these problems by allowing code to **explicitly request or reject compiler-generated functions**. For example:

```cpp
class Magic {
    public:
    Magic() = default; // explicitly use the compiler-generated constructor
    Magic& operator=(const Magic&) = delete; // explicitly reject the compiler-generated assignment operator
    Magic(int magic_number);
}
```

#### Explicit Virtual Function Overrides

In traditional C++, accidental virtual-function overriding is easy. For example:

```cpp
struct Base {
    virtual void foo();
}
struct SubClass: Base {
    void foo();
}
```

`SubClass::foo` may not be an intentional override; the programmer may simply have added a function with the same name by coincidence. Another possible case is that, after a base-class virtual function is removed, an old function in the derived class no longer overrides anything and silently becomes an ordinary member function. That can have serious consequences.

C++11 introduced the `override` and `final` keywords to prevent these cases.

**override**

When overriding a virtual function, the `override` keyword explicitly tells the compiler that an override is intended.

The compiler checks whether such a virtual function exists in the base class; otherwise the code fails to compile:

```cpp
struct Base {
    virtual void foo(int);
};
struct SubClass: Base {
    virtual void foo(int) override; // valid
    virtual void foo(float) override; // invalid: the base class has no such virtual function
};
```

**final**

`final` was introduced to prevent a class from **being further inherited** and to **stop further overriding of a virtual function**.

```cpp
struct Base {
    virtual void foo() final;
};

struct SubClass1 final: Base {
}; // valid

struct SubClass2 : SubClass1 {
}; // invalid: SubClass1 is final

struct SubClass3: Base {
    void foo(); // invalid: foo is final
};
```

#### Strongly Typed Enumerations

In traditional C++, enumeration types are not fully type-safe.

Enumeration values are treated like integers, which can allow direct comparisons between completely different enum types. Compilers provide some checks, but not all problematic cases are rejected. Also, **enumerator names from different enum types in the same namespace cannot be repeated**, which is often undesirable.

C++11 introduced enumeration classes, declared with `enum class` syntax:

```cpp
enum class new_enum : unsigned int {
    value1,
    value2,
    value3 = 100,
    value4 = 100
};
```

Enums defined this way are type-safe:

First, they cannot be implicitly converted to integers. They also cannot be compared directly with integer values, and enumerators from different enum classes cannot be compared directly.

However, enumerators from the same enum class can be compared, even if two names have the same assigned value:

```cpp
if (new_enum::value3 == new_enum::value4) {
    ...
}
```

In this syntax, the **colon followed by a type keyword** after the enum name specifies the underlying type of the enumerators, here `unsigned int`. This allows us to assign explicit enumerator values. If no underlying type is specified, `int` is used by default.

When we want to obtain the numeric value of an enumerator, we must perform an explicit cast. Alternatively, we can overload the `<<` operator for output. The following snippet is useful:

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

With that overload, the following code can compile:

```cpp
std::cout << new_enum::value3 << std::endl
```

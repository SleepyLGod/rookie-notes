# Modern C++ Templates

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

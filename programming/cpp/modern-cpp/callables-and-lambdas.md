# Callables and Lambdas

### Generic Lambda

In C++11, the `auto` keyword could not be used in a function parameter list, because such syntax would conflict with the role of templates. However, a Lambda expression is not an ordinary function. Without an explicit parameter type, a C++11 Lambda expression could not be templated in the same way.

Fortunately, this limitation only exists in C++11.

Starting from C++14, Lambda function parameters can use the `auto` keyword, which gives Lambda expressions generic behavior:

```cpp
auto add = [](auto x, auto y) {
    return x + y;
}
add(1, 2);
```

### Function object wrapper

#### std::function

The essence of a Lambda expression is an object, called a closure object, whose class type is similar to a **function object type**. That class type is called the closure type.

When the capture list of a Lambda expression is empty, the closure object can also be converted to a function pointer value and passed around. For example:

```cpp
#include <iostream>

using foo = void(int); // Define a function type. See the alias syntax in the previous section for using.
void functional(foo f) { // The function type foo in the parameter list is treated as the decayed function pointer type foo*.
    f(1); // Call the function through the function pointer.
}

int main() {
    auto f = [](int value) {
        std::cout << value << std::endl;
    };
    functional(f); // Pass the closure object, implicitly converted to a function pointer value of type foo*.
    f(1); // Call the Lambda expression directly.
    return 0;
}
```

The code above shows two different calling forms: one passes the Lambda as a function type and calls it through that interface, while the other calls the Lambda expression directly.

In **C++11**, these concepts were unified under the idea of **callable types**, meaning the types of objects that can be invoked. `std::function` is the standard wrapper introduced for this purpose.

C++11 `std::function` is a **general-purpose, polymorphic function wrapper**. Its instances can store, copy, and call any callable target entity. It is also a type-safe wrapper around existing callable entities in C++; by comparison, raw function-pointer calls are less expressive and easier to misuse. In other words, `std::function` is a **container for functions**. Once we have such a container, functions and function pointers can be handled more conveniently as objects. For example:

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

#### std::bind and std::placeholder

`std::bind` is used to **bind arguments for a function call**.

The problem it solves is this: sometimes we cannot obtain all arguments needed to call a function at once. With `std::bind`, we can bind some arguments to the function in advance, produce a new callable object, and complete the call later when the remaining arguments are available. For example:

```cpp
int foo(int a, int b, int c) {...}
int main() {
    // Bind arguments 1 and 2 to function foo,
    // but use std::placeholders::_1 as a placeholder for the first argument.
    auto bindFoo = std::bind(foo, std::placeholders::_1, 1,2);
    // Now, when calling bindFoo, only the first argument needs to be provided.
    bindFoo(1);
}
```

**Tip:** pay attention to the usefulness of the `auto` keyword. Sometimes we may not be familiar with a function's return type, but `auto` lets us avoid spelling that type manually.

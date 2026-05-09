# Value Categories and Move Semantics

### Rvalues

C++11 refined the concept of rvalues in order to introduce powerful rvalue references. Rvalues are further divided into prvalues and xvalues.

**Prvalue** means "pure rvalue." It is either a literal such as `10` or `true`, or an expression whose evaluation result is equivalent to a literal or an unnamed temporary object, such as `1 + 2`. Temporaries returned by non-reference return types, temporaries produced by arithmetic expressions, ordinary literals, and lambda expressions are all prvalues.

One detail is important: literals are prvalues except for string literals. A string literal is an lvalue whose type is an array of `const char`. For example:

```cpp
#include <type_traits>

int main() {
    // valid: "01234" has type const char [6], so it is an lvalue
    const char (&left)[6] = "01234";

    // assertion succeeds: the type is indeed const char [6].
    // Note that decltype(expr) returns an lvalue reference when expr is an lvalue
    // and is not an unparenthesized id-expression or class-member access.
    static_assert(std::is_same<decltype("01234"), const char(&)[6]>::value, "");

    // invalid: "01234" is an lvalue and cannot bind to an rvalue reference
    // const char (&&right)[6] = "01234";
}
```

However, arrays can be implicitly converted to the corresponding pointer type. The result of a conversion expression, if it is not an lvalue reference, is an rvalue: an xvalue if it is an rvalue reference, otherwise a prvalue. For example:

```cpp
const char*   p   = "01234";  // valid: "01234" is implicitly converted to const char*
const char*&& pr  = "01234";  // valid: the conversion result is a const char* prvalue
// const char*& pl = "01234"; // invalid: there is no const char* lvalue here
```

**Xvalue** means "expiring value." It is a concept introduced by C++11 together with rvalue references. In traditional C++, prvalue and rvalue were effectively the same concept. An xvalue is a value that is near the end of its lifetime but can still be moved from.

Xvalues can be difficult to understand, so consider this code:

```cpp
std::vector<int> foo() {
    std::vector<int> temp = {1, 2, 3, 4};
    return temp;
}

std::vector<int> v = foo();
```

Under a traditional mental model, the return value `temp` is created inside `foo` and then assigned to `v`. When `v` obtains the object, the whole `temp` would be copied and then `temp` would be destroyed. If `temp` is large, this creates substantial extra cost, which is one of the long-standing complaints about older C++ styles.

In the last line, `v` is an lvalue, and the value returned by `foo()` is an rvalue, specifically a prvalue in the expression model.

However, `v` can be accessed by other variables, while the return value produced by `foo()` is a temporary. Once it has been used to initialize `v`, the temporary is immediately destroyed and cannot be accessed or modified directly.&#x20;

The xvalue category captures this behavior: **a temporary-like value that can be identified and moved from**.

After C++11, the compiler helps here. In a return statement like this, the local object `temp` can be treated as a movable value, conceptually similar to `static_cast<std::vector<int>&&>(temp)`, so `v` can be initialized by moving the local return value from `foo`. This is part of the move semantics discussed below.

To obtain an xvalue explicitly, we use an **rvalue reference: `T&&`**.&#x20;

Binding a temporary to an rvalue reference can extend the temporary's lifetime; as long as the reference variable remains alive, the temporary remains alive.

C++11 provides `std::move`, which **unconditionally casts its argument to an rvalue expression**. With it, we can conveniently obtain a movable expression from an lvalue:

```cpp
void reference(std::string& str) {
    std::cout << "lvalue" << std::endl;
}
void reference(std::string&& str) {
    std::cout << "rvalue" << std::endl;
}
int main() {
    std::string lv1 = "string,"; // lv1 is an lvalue
    // std::string&& r1 = lv1; // invalid: an rvalue reference cannot bind to an lvalue
    std::string&& rv1 = std::move(lv1); // valid: std::move casts an lvalue to an rvalue expression
    std::cout << rv1 << std::endl; // string,

    const std::string& lv2 = lv1 + lv1; // valid: a const lvalue reference can extend a temporary's lifetime
    // lv2 += "Test"; // invalid: a const reference cannot be modified
    std::cout << lv2 << std::endl; // string,string,

    std::string&& rv2 = lv1 + lv2; // valid: an rvalue reference extends the temporary object's lifetime
    rv2 += "Test"; // valid: a non-const reference can modify the temporary
    std::cout << rv2 << std::endl; // string,string,string,Test

    reference(rv2); // prints lvalue

    return 0;
} 
```

Traditional C++ models object copying through copy constructors and assignment operators. To transfer resources, callers often had to copy first and then destroy the old object, or manually implement a custom transfer interface. Conceptually, moving house should mean taking the existing belongings to the new place, not buying duplicates for the new place and then throwing away everything in the old one.

Traditional C++ did not clearly separate **move** and **copy** operations, which led to unnecessary data copies and wasted time and space. Rvalue references solve this conceptual confusion. For example:

```cpp
#include <iostream>
class A {
public:
    int *pointer;
    A():pointer(new int(1)) {
        std::cout << "construct " << pointer << std::endl;
    }
    A(A& a):pointer(new int(*a.pointer)) {
        std::cout << "copy " << pointer << std::endl;
    } // unnecessary object copy
    A(A&& a):pointer(a.pointer) {
        a.pointer = nullptr;
        std::cout << "move " << pointer << std::endl;
    }
    ~A(){
        std::cout << "destruct " << pointer << std::endl;
        delete pointer;
    }
};
// prevent the compiler from optimizing the example away
A return_rvalue(bool test) {
    A a, b;
    if(test) return a; // conceptually similar to static_cast<A&&>(a);
    else return b;     // conceptually similar to static_cast<A&&>(b);
}
int main() {
    A obj = return_rvalue(false);
    std::cout << "obj:" << std::endl;
    std::cout << obj.pointer << std::endl;
    std::cout << *obj.pointer << std::endl;
    return 0;
}
```

In the code above:

1. Two `A` objects are first constructed inside `return_rvalue`, so two constructor messages are printed.
2. When the function returns, it produces an xvalue-like result that can bind to `A`'s move constructor (`A(A&&)`). The pointer from the returned object is transferred into `obj`, while the source object's pointer is set to `nullptr`, preventing that memory from being deleted by the moved-from object.

This avoids an unnecessary copy construction and improves performance. Now look at an example involving the standard library:

```cpp
#include <iostream> // std::cout
#include <utility> // std::move
#include <vector> // std::vector
#include <string> // std::string

int main() {

    std::string str = "Hello world.";
    std::vector<std::string> v;

    // uses push_back(const T&), which copies
    v.push_back(str);
    // prints "str: Hello world."
    std::cout << "str: " << str << std::endl;

    // uses push_back(T&&), which moves rather than copies
    // the string can be moved into the vector, so std::move is often used to reduce copy cost
    // after this operation, str remains valid, but its value is unspecified
    v.push_back(std::move(str));
    // commonly prints "str: ", but the standard does not require it to be empty
    std::cout << "str: " << str << std::endl;

    return 0;
}
```

#### Perfect Forwarding

As mentioned above, a named rvalue reference is itself an lvalue expression. This creates a problem for parameter forwarding:

```cpp
void reference(int& v) {
    std::cout << "lvalue" << std::endl;
}
void reference(int&& v) {
    std::cout << "rvalue" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "ordinary forwarding:";
    reference(v); // always calls reference(int&)
}
int main() {
    std::cout << "passing rvalue:" << std::endl;
    pass(1); // 1 is an rvalue, but the output is lvalue

    std::cout << "passing lvalue:" << std::endl;
    int l = 1;
    pass(l); // l is an lvalue, output is lvalue

    return 0;
}

```

For `pass(1)`, the argument is an rvalue, but the parameter `v` is a named reference, so the expression `v` is an lvalue. Therefore `reference(v)` calls `reference(int&)` and prints "lvalue." For `pass(l)`, `l` is an lvalue, so why can it be passed successfully to `pass(T&&)`?

This is based on **reference collapsing rules**. In older C++, references to references were not directly part of everyday source syntax. With rvalue references and template deduction, C++ defines reference collapsing rules that allow references to combine with other references. The result follows these rules:

| Function parameter type | Argument reference type | Deduced function parameter type |
| :----: | :----: | :-------: |
|   T&   |   lvalue reference  |     T&    |
|   T&   |   rvalue reference  |     T&    |
|   T&&  |   lvalue reference  |     T&    |
|   T&&  |   rvalue reference  |    T&&    |

Therefore, using `T&&` in a template function does not necessarily mean an rvalue reference after deduction. When an lvalue is passed, the parameter type collapses to an lvalue reference. More precisely, **regardless of the reference form in the template parameter, the deduced result becomes an rvalue reference only when the argument is an rvalue reference; otherwise it collapses to an lvalue reference**. This is why `v` can be passed successfully when the original argument is an lvalue.

Perfect forwarding is built on these rules. It means preserving the original value category while forwarding parameters: lvalues stay lvalues, and rvalues stay rvalues. To solve this problem, use `std::forward` when forwarding parameters:

```cpp
#include <iostream>
#include <utility>
void reference(int& v) {
    std::cout << "lvalue reference" << std::endl;
}
void reference(int&& v) {
    std::cout << "rvalue reference" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "       ordinary forwarding: ";
    reference(v);
    std::cout << "        std::move forwarding: ";
    reference(std::move(v));
    std::cout << "     std::forward forwarding: ";
    reference(std::forward<T>(v));
    std::cout << "static_cast<T&&> forwarding: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << "passing rvalue:" << std::endl;
    pass(1);

    std::cout << "passing lvalue:" << std::endl;
    int v = 1;
    pass(v);

    return 0;
}
```

The output is:

```cpp
passing rvalue:
       ordinary forwarding: lvalue reference
        std::move forwarding: rvalue reference
     std::forward forwarding: rvalue reference
static_cast<T&&> forwarding: rvalue reference
passing lvalue:
       ordinary forwarding: lvalue reference
        std::move forwarding: rvalue reference
     std::forward forwarding: lvalue reference
static_cast<T&&> forwarding: lvalue reference

```

Whether the original argument is an lvalue or an rvalue, ordinary forwarding passes the named parameter as an lvalue. `std::move` always casts that named parameter to an rvalue expression, so it calls `reference(int&&)` and prints "rvalue reference."

Only `std::forward` avoids extra copying while **perfectly forwarding** the function argument to the internal call.

Like `std::move`, `std::forward` does not move data by itself. `std::move` simply casts an expression to an rvalue. `std::forward` also performs a cast, but the target type depends on template deduction. In the examples above, `std::forward<T>(v)` behaves like `static_cast<T&&>(v)`.

A natural question is how one expression can produce the appropriate value category for both cases. To see this, look briefly at the implementation mechanism of `std::forward`, which has two overloads:

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

In this implementation, `std::remove_reference` removes references from a type, and `std::is_lvalue_reference` checks whether type deduction produced the expected form. The second overload of `std::forward` rejects an invalid attempt to forward an rvalue as an lvalue reference, reflecting the reference-collapsing rules.

When `std::forward` receives an lvalue in a forwarding context, `_Tp` is deduced in a way that makes the return expression an lvalue. When it receives an rvalue, `_Tp` is deduced so that, after reference collapsing, the return expression is an rvalue. The key idea is that `std::forward` deliberately exploits the differences produced by template type deduction.

This also answers a common question: why is `auto&&` often the safest form in generic loop code? Because when `auto` is deduced from different lvalue or rvalue expressions, combining it with `&&` gives the reference-collapsing behavior needed for perfect forwarding.

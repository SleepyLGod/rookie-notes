---
description: Reference https://changkun.de/modern-cpp/en-us/05-pointers/
---

# Smart Pointers

### Introduction

Forgetting to call `delete`, or calling `delete` in the wrong place, can be disastrous.

Automatic memory release is best handled by classes and RAII.

That is why smart pointers such as <mark style="color:purple;background-color:blue;">`auto_ptr`</mark>, <mark style="color:purple;background-color:blue;">`unique_ptr`</mark>, and <mark style="color:purple;background-color:purple;">`shared_ptr`</mark> were introduced.

<mark style="color:purple;">**The basic idea is to wrap a raw pointer in a class object. The wrapper class is a template so it can support different pointed-to types, and its destructor calls `delete` to release the memory owned by the pointer.**</mark>

Now consider **reference counting**:

> The basic idea is to maintain a reference count for dynamically allocated objects. Whenever another reference to the same object is added, the reference count increases by one. Whenever a reference is removed, the count decreases by one. When the reference count reaches zero, the heap memory pointed to by the object is automatically deleted.

The C++ standard library provides smart pointers such as `std::shared_ptr`, `std::unique_ptr`, and `std::weak_ptr`. To use them, include the `<memory>` header.

The `auto_ptr` template was the C++98 solution. C++11 deprecated it and introduced the modern alternatives. `auto_ptr` was removed in C++17 and should not be used in new code.

Smart pointer classes generally provide an **explicit constructor** that takes a raw pointer. For example, the old `auto_ptr` class template looked like this:

```cpp
template<class T>
class auto_ptr {
  explicit auto_ptr(X* p = 0) ; 
  ...
}
```

Therefore, a raw pointer cannot be implicitly converted to a smart pointer object. The conversion must be **explicit**:

```cpp
shared_ptr<double> pd; 
double *p_reg = new double;
pd = p_reg;                               // not allowed (implicit conversion)
pd = shared_ptr<double>(p_reg);           // allowed (explicit conversion)
shared_ptr<double> pshared = p_reg;       // not allowed (implicit conversion)
shared_ptr<double> pshared(p_reg);        // allowed (explicit conversion)
```

One thing should be **avoided** for all owning smart pointers:

```cpp
string vacation("I wandered lonely as a cloud.");
shared_ptr<string> pvac(&vacation);   // No
```

`vacation` is a stack object, not heap memory allocated by `new`. When `pvac` expires, the program will apply `delete` to non-heap memory, which is incorrect.

### `std::shared_ptr`

`std::shared_ptr` records how many `shared_ptr` instances **jointly point to the same object**. This removes the need for explicit `delete`; when the reference count becomes zero, the object is automatically deleted.

However, directly constructing a `std::shared_ptr` from `new` still leaves an awkward asymmetry in the code.

**`std::make_shared`** removes the explicit use of `new`. It allocates and constructs the object from the provided arguments and returns a `std::shared_ptr` to that object type. For example:

```cpp
#include<iostream>
#include<memory>
void foo(std::shared_ptr<int> i) {
    (*i)++;
}
int main() {
    // auto pointer = new int(10); // illegal, no direct assignment
    // Constructed a std::shared_ptr
    auto pointer = std::make_shared<int>(10);
    foo(pointer);
    std::cout << *pointer << std::endl; // 11
    // The shared_ptr will be destructed before leaving the scope
    return 0;
}
```

`std::shared_ptr` can obtain the raw pointer through **`get()`**, decrease the reference count through **`reset()`**, and inspect the reference count through **`use_count()`**. For example:

```cpp
auto pointer = std::make_shared<int>(10);
auto pointer2 = pointer; // reference count + 1
auto pointer3 = pointer; // reference count + 1
int *p = pointer.get();  // this does not increase the reference count
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 3
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 3
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 3

pointer2.reset();
std::cout << "reset pointer2:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 2
std::cout << "pointer2.use_count() = "
          << pointer2.use_count() << std::endl;           // pointer2 has been reset; 0
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 2
pointer3.reset();
std::cout << "reset pointer3:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 1
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 0
std::cout << "pointer3.use_count() = "
          << pointer3.use_count() << std::endl;           // pointer3 has been reset; 0

```

### **`std::unique_ptr`**

`std::unique_ptr` is an exclusive-ownership smart pointer. It prevents other smart pointers from sharing ownership of the same object, improving ownership safety:

```cpp
std::unique_ptr<int> pointer = std::make_unique<int>(10); // make_unique was introduced in C++14
std::unique_ptr<int> pointer2 = pointer; // invalid
```

`make_unique` is not complicated. C++11 did not provide `std::make_unique`, so it can be implemented manually:

```cpp
template<typename T, typename ...Args>
std::unique_ptr<T> make_unique( Args&& ...args ) {
  return std::unique_ptr<T>( new T( std::forward<Args>(args)... ) );
}
```

As for why it was not provided in C++11, Herb Sutter mentioned in his [blog](https://herbsutter.com/gotw/\_102/) that it was essentially forgotten. See [**this discussion**](https://stackoverflow.com/questions/12580432/why-does-c11-have-make-shared-but-not-make-unique).

Because ownership is exclusive, `unique_ptr` is not copyable. However, ownership can be transferred to another `unique_ptr` with `std::move`:

```cpp
#include <iostream>
#include <memory>

struct Foo {
    Foo() { std::cout << "Foo::Foo" << std::endl; }
    ~Foo() { std::cout << "Foo::~Foo" << std::endl; }
    void foo() { std::cout << "Foo::foo" << std::endl; }
};

void f(const Foo &) {
    std::cout << "f(const Foo&)" << std::endl;
}

int main() {
    std::unique_ptr<Foo> p1(std::make_unique<Foo>());
    // p1 is not empty; this prints output
    if (p1) {
        p1->foo();
    }
    {
        std::unique_ptr<Foo> p2(std::move(p1));
        // p2 is not empty; this prints output
        f(*p2);
        // p2 is not empty; this prints output
        if(p2) {p2->foo();}
        // p1 is empty; this prints nothing
        if(p1) {p1->foo();}
        p1 = std::move(p2);
        // p2 is empty; this prints nothing
        if(p2) {p2->foo();}
        std::cout << "p2 is destroyed" << std::endl;
    }
    // p1 is not empty; this prints output
    if (p1) {p1->foo();}
    // The Foo instance is destroyed when leaving the scope
}
```

### **`std::weak_ptr`**

If we look more carefully at `std::shared_ptr`, there is still a case where resources cannot be released: cyclic ownership. Consider the following example:

```cpp
struct A;
struct B;

struct A {
    std::shared_ptr<B> pointer;
    ~A() {
        std::cout << "A is destroyed" << std::endl;
    }
};
struct B {
    std::shared_ptr<A> pointer;
    ~B() {
        std::cout << "B is destroyed" << std::endl;
    }
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->pointer = b;
    b->pointer = a;
}
```

The result is that neither `A` nor `B` is destroyed. The reason is that the internal `pointer` members of `a` and `b` also reference `a` and `b`, so the reference count of both objects becomes 2. When the local smart pointers `a` and `b` leave scope, their destruction only decreases those reference counts by one. The memory areas pointed to by the `A` and `B` objects therefore still have non-zero reference counts, but the outside program can no longer reach them. This creates a memory leak, as shown below:

![](<../../.gitbook/assets/image (6).png>)

The solution is to use the weak-reference pointer `std::weak_ptr`. A `std::weak_ptr` is a weak reference, whereas `std::shared_ptr` is a strong owning reference. A weak reference does not increase the reference count. After replacing one side of the ownership cycle with a weak reference, the final release process becomes:

![](<../../.gitbook/assets/image (2) (3).png>)

In the diagram above, only `B` remains at the last step, and no owning smart pointer references it anymore. Therefore, that memory resource is also released.

`std::weak_ptr` does not provide `operator*` or `operator->`, so it cannot directly operate on the resource. It can be used to check whether the corresponding `std::shared_ptr`-managed object still exists. Its `expired()` method returns `false` when the resource has not been released, and returns `true` otherwise. It can also be used to obtain a `std::shared_ptr` that points to the original object: `lock()` returns such a `std::shared_ptr` when the original object is still alive, allowing the resource to be accessed; otherwise it returns `nullptr`.

### **How to choose?**

If a program needs **multiple pointers to the same object**, choose `shared_ptr`. Typical cases include:

* There is an array of pointers, and additional helper pointers are used to mark specific elements, such as the largest or smallest element.
* Two objects both contain pointers to a third object.
* An STL container stores pointers. Many STL algorithms require copy and assignment operations. These operations are supported by `shared_ptr`, but not by `unique_ptr` in copy contexts, and `auto_ptr` has problematic ownership-transfer behavior. If the compiler does not provide `shared_ptr`, the `shared_ptr` implementation from **Boost** can be used.

If the program does not need multiple pointers to the same object, use `unique_ptr`.

If a function allocates memory with `new` and returns a pointer to that memory, declaring its return type as `unique_ptr` is a good choice. Ownership is transferred to the `unique_ptr` that receives the return value, and that smart pointer becomes responsible for calling `delete`. A `unique_ptr` can also be stored in an STL container, **as long as no algorithm tries to copy or assign one `unique_ptr` to another non-temporary `unique_ptr`**. For example:

```cpp
unique_ptr<int> make_int(int n) {
    return unique_ptr<int>(new int(n));
}
void show(unique_ptr<int> &p1) {
    cout << *p1 << ' ';
}
int main() {
    ...
    vector<unique_ptr<int> > vp(size);
    for(int i = 0; i < vp.size(); i++) {
        vp[i] = make_int(rand() % 1000); // copy temporary unique_ptr
    }
    vp.push_back(make_int(rand() % 1000)); // ok because arg is temporary
    for_each(vp.begin(), vp.end(), show); // use for_each()
    ...
}
```

The `push_back` call is valid because `make_int()` **returns a temporary `unique_ptr`**, and that temporary is moved into a `unique_ptr` stored in `vp`. However, if `show()` received its argument **by value instead of by reference**, the `for_each()` call would be illegal. It would try to initialize `p1` from a non-temporary `unique_ptr` inside `vp`, which is not allowed. As noted above, the compiler detects incorrect attempts to copy `unique_ptr`.

When a `unique_ptr` is an rvalue, it can be used to construct a `shared_ptr`. This follows the same ownership-transfer condition required when moving a `unique_ptr`. As before, in the following code `make_int()` returns `unique_ptr<int>`:

```cpp
unique_ptr<int> pup(make_int(rand() % 1000));   // ok
shared_ptr<int> spp(pup);                       // not allowed, pup as lvalue
shared_ptr<int> spr(make_int(rand() % 1000));   // ok
```

The `shared_ptr` template provides a constructor that can convert an rvalue `unique_ptr` into a `shared_ptr`. The `shared_ptr` takes over ownership of the object that was previously owned by the `unique_ptr`.

In old C++ code, `auto_ptr` could be used in some situations where exclusive ownership was needed, but `unique_ptr` is the better and modern choice. If the compiler does not provide `unique_ptr`, consider **Boost's `scoped_ptr`**, which has similar single-owner intent.

---
description: Notes adapted from several official and reference explanations
---

# `typename` or `class`?

`typename` is used to state that a `qualified name` denotes a type. For example:

```cpp
template<class C> void f(C& rc) { 
        typename C::iterator i = rc.begin();
        //   ... 
} 
```

The compiler does not know the definition of `C` while parsing the template, so it does not know what `C::iterator` denotes. Therefore **`typename`** is required to tell the compiler that it is a type.

In **`template<typename T, template<typename T> class S>`**, `S` is itself a template parameter.

C++ uses `class` in the template-template parameter syntax here, so replacing this `class` with `typename` is not valid in older standards. For example:

```cpp
// The implementation of class A
class A { 
  int a;
} 
// Another possible implementation of class A
class A { 
  typedef int a; 
} 
```

In the second definition, `typename A::a i` means that **`A::a` is a type**, not a data member.

And `i` is **an instance of type `A::a`**.

**In template type-parameter declarations, `class` and `typename` are equivalent.** When templates were first designed, `class` was reused to avoid adding a new keyword. Later, however, C++ still needed the `typename` keyword for cases like the one above.

**`class` can define classes and can also introduce template type parameters, while `typename` is used to introduce type parameters or disambiguate dependent type names.**

```cpp
template<class T> class Widget; // uses "class"
template<typename T> class Widget; // uses "typename"
```

**When declaring a template type parameter, `class` and `typename` mean exactly the same thing.**

Some programmers prefer to always use `class` because it is shorter to type.

Others prefer `typename` because it suggests that the parameter does not have to be a class type.

A few developers use `typename` whenever any type is allowed and reserve `class` for cases where only user-defined types are intended.

From the C++ language's point of view, however, `class` and `typename` mean exactly the same thing when declaring a template type parameter.

However, C++ does not always treat `class` and `typename` as equivalent. **Sometimes you must use `typename`.**

To understand why, we need to discuss two kinds of names that appear inside templates.

**Suppose** we have a function template that accepts an STL-compatible container whose elements can be assigned to `int`. Suppose further that the function simply prints the value of the second element. This is a poorly designed function implemented in a clumsy way, and as written below it does not even compile. Put that aside for the moment; the example is useful for exposing the issue:

```cpp
template<typename C> // print 2nd element in
void print2nd(const C& container) // container;
{
    // this is not valid C++!
    if (container.size() >= 2) {
        C::const_iterator iter(container.begin()); // get iterator to 1st element
        ++iter; // move iter to 2nd element
        int value = *iter; // copy that element to an int
        std::cout << value; // print the int
    }
}
```

There are two `local variables` in this function: `iter` and `value`.

*   The type of **`iter`** is `C::const_iterator`, a type that depends on template parameter `C`. A name inside a template that depends on a template parameter is called a **dependent name**.

    When a dependent name is nested inside a class or class-like scope, it is a **nested dependent name**.

    `C::const_iterator` is a nested dependent name. More specifically, it is a **nested dependent type name**, meaning a nested dependent name that denotes a type.
*   The other local variable, **`value`**, has type `int`. `int` does not depend on any template parameter. Such names are called **non-dependent names**.

    **Nested dependent names create parsing difficulties.**

    For example, suppose we start `print2nd` in an even worse way:

```cpp
template<typename C>
void print2nd(const C& container) {
	C::const_iterator * x;
	...
}
```

This appears to declare `x` as a local variable that is a pointer to `C::const_iterator`.

But it only looks that way because we assume `C::const_iterator` is a type.

**What if `C::const_iterator` is not a type?**

What if `C` has a static data member named `const_iterator`? What if `x` is the name of a global variable?

In that case, the code above is not a local-variable declaration. It becomes `C::const_iterator * x`, a multiplication expression. That sounds silly, but it is possible, and C++ parser writers must account for all valid inputs, including surprising ones.

Until `C` is known, there is no way to know whether `C::const_iterator` is a type. When the template `print2nd` is parsed, `C` is not yet known.

C++ resolves this ambiguity with a rule: **when the parser encounters a nested dependent name inside a template, it assumes that the name is not a type unless you tell it otherwise.**

By default, nested dependent names are not assumed to be types. There is one exception to this rule, described below.

With that in mind, look again at the beginning of `print2nd`:

```cpp
template<typename C>
void print2nd(const C& container) {
    if (container.size() >= 2) {
    	C::const_iterator iter(container.begin()); // this name is assumed to
    	... // not be a type
```

Now it should be clear why this is not valid C++:

The declaration of `iter` only makes sense if `C::const_iterator` is a type. But we did not tell C++ that it is, so C++ assumes that it is not. To change that, we must tell C++ that `C::const_iterator` is a type by placing `typename` immediately before it:

```cpp
template<typename C> // this is valid C++
void print2nd(const C& container) {
    if (container.size() >= 2) {
        typename C::const_iterator iter(container.begin());
        ...
    }
}
```

The general rule is simple: **whenever you refer to a nested dependent type name inside a template, put the word `typename` immediately before it.** Again, there is an exception described below.

**`typename` should only be used to identify nested dependent type names;** other names should not be prefixed with it.

For example, here is a function template that takes a container and an iterator into that container:

```cpp
template<typename C> // typename allowed (as is "class")
void f(const C& container, /* typename not allowed */ typename C::iterator iter /* typename required */);
```

`C` is not a nested dependent type name. It is not nested inside something dependent on a template parameter, so it must not be prefixed with `typename` when declaring `container`.

But `C::iterator` is a nested dependent type name, so it must be prefixed with `typename`.

The exception to the rule that "`typename` must precede nested dependent type names" is this: **`typename` must not be used before a nested dependent type name in a base-class list, or before a nested dependent type name used as a base-class identifier in a member-initializer list**. For example:

```cpp
template<typename T>
class Derived: public Base<T>::Nested {
// base class list: typename is not
public: // allowed
    explicit Derived(int x)
    : Base<T>::Nested(x) // base class identifier in a member
    {
        // initializer list: typename is not allowed
        
        typename Base<T>::Nested temp; // nested dependent type name used
        ... // outside a base class list and not as
    } // a base class identifier in a member
    ... // initializer list: typename is required
};
```

This inconsistency is annoying, but after some experience you rarely notice it.

Consider one final `typename` example, because it is representative of real code.

Suppose we are writing a function template that takes an iterator, and we want to make a local copy `temp` of the object pointed to by the iterator. We can write:

```cpp
template<typename IterT>
void workWithIterator(IterT iter) {
    typename std::iterator_traits<IterT>::value_type temp(*iter);
    ...
}
```

Do not be intimidated by **`std::iterator_traits<IterT>::value_type`**. It is just a use of a standard traits class. In C++ terms, it means "**the type of thing pointed to by objects of type `IterT`**." This statement declares a local variable `temp` whose type is the same as the type pointed to by `IterT` objects, and initializes `temp` with the object pointed to by `iter`.

If `IterT` is `vector<int>::iterator`, `temp` has type `int`.

If `IterT` is `list<string>::iterator`, `temp` has type `string`. Because `std::iterator_traits<IterT>::value_type` is a nested dependent type name, since `value_type` is nested inside `iterator_traits<IterT>` and `IterT` is a template parameter, it must be prefixed with `typename`.

If reading `std::iterator_traits<IterT>::value_type` is unpleasant, introduce a shorter name for it. If, like most programmers, you do not want to type it repeatedly, create a `typedef`. For traits member names such as `value_type`, **a common convention is to give the typedef the same name as the traits member**, so a local typedef is often written like this:

```cpp
template<typename IterT>
void workWithIterator(IterT iter) {
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
    ...
}
```

Many programmers initially find the phrase `typedef typename` awkward, but it is a natural consequence of the nested-dependent-type-name rule. You get used to it quickly, especially because typing `typename std::iterator_traits<IterT>::value_type` repeatedly is much worse.

As a final note, compilers have historically differed in how strictly they enforce the rules around `typename`.

Some compilers accept code where a required `typename` is missing.

Some compilers accept code where `typename` is present even though it is not allowed.

A few, usually older compilers, reject `typename` even where it is required.

This means the interaction between `typename` and nested dependent type names can cause minor portability issues.

**Things to Remember**&#x20;

**1. In template type-parameter declarations, `class` and `typename` are interchangeable.**&#x20;

**2. Use `typename` to identify nested dependent type names, except in base-class lists and when the name is used as a base-class identifier in a member-initializer list.**

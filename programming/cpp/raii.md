# RAII

RAII is short for <mark style="color:purple;">**resource acquisition is initialization**</mark>.

It is a design idea proposed by Bjarne Stroustrup, the creator of C++. Its core is to bind resources to object lifetimes: object construction acquires resources, and object destruction releases them. Under RAII, C++ lifts low-level resource management into higher-level object lifetime management.

<mark style="color:purple;background-color:blue;">**`->`**</mark> [<mark style="color:purple;background-color:blue;">**`Smart pointers`**</mark>](smart-pointers.md) use RAII to ensure that resources are correctly initialized and released. A smart pointer is essentially an object, but it behaves like a pointer.

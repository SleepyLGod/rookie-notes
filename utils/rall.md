# 😁 RALL

RAII ， <mark style="color:purple;">**resource acquisition is initialization**</mark> 的缩写，意为“资源获取即初始化”。

它是 C++ 之父 Bjarne Stroustrup 提出的设计理念，其核心是把资源和对象的生命周期绑定，对象创建获取资源，对象销毁释放资源。在 RAII 的指导下，C++ 把底层的资源管理问题提升到了对象生命周期管理的更高层次。

<mark style="color:purple;background-color:blue;">**`🤜`**</mark>[<mark style="color:purple;background-color:blue;">**`智能指针`**</mark>](smart-pointers.md)就是通过使用RAII技术，来保证资源正确的初始化和释放，实质上是一个对象，行为上表现得却像一个指针


# 🥲 智能指针

忘了加delete？不在合适的地方加入delete？那将是一场灾难！

自动释放内存，只有类可以做到😭

于是智能指针<mark style="color:purple;background-color:blue;">`auto_ptr`</mark>、<mark style="color:purple;background-color:blue;">`unique_ptr`</mark>和<mark style="color:purple;background-color:purple;">`shared_ptr`</mark>来了！

<mark style="color:purple;">**将基本类型指针封装为类对象指针（这个类肯定是个模板，以适应不同基本类型的需求），并在析构函数里编写delete语句删除指针指向的内存空间。**</mark>

STL一共给我们提供了四种智能指针：auto\_ptr、unique\_ptr、shared\_ptr和weak\_ptr（暂不讨论）。&#x20;

模板`auto_ptr`是C++98提供的解决方案，C+11已将将其摒弃，并提供了另外两种解决方案。然而，虽然auto\_ptr被摒弃，但它已使用了好多年：同时，如果您的编译器不支持其他两种解决力案，auto\_ptr将是唯一的选择。

<mark style="color:purple;">****</mark>


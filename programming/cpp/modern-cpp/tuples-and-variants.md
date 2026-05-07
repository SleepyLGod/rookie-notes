# Tuples and Variants

### 元组

纵观传统 C++ 中的容器，除了 `std::pair` 外， 似乎没有现成的结构能够用来存放不同类型的数据（通常我们会自己定义结构）。 但 `std::pair` 的缺陷是显而易见的，只能保存两个元素。

#### 基本操作

关于元组的使用有三个核心的函数：

1. `std::make_tuple`: 构造元组
2. `std::get`: 获得元组某个位置的值
3. `std::tie`: 元组拆包

```cpp
#include<tuple>
#include<iostream>

auto get_student(int id) {
    // 返回类型被推断为 std::tuple<double, char, std::string>
    if (id == 0) {
        return std::make_tuple(3.8, 'A', "zhangsan");
    } else if (id == 1) {
        return std::make_tuple(2.9, 'C', "李四");
    } else if (id == 2) {
        return std::make_tuple(1.7, 'D', "王五");
    }
    return std::make_tuple(0.0, 'D', "null");
    // 如果只写 0 会出现推断错误, 编译失败!!!!!
}

int main() {
    auto student = get_student(0);
    std:cout << "ID: 0, "
    << "GPA: " << std::get<0>(student) << ", "
    << "成绩: " << std::get<1>(student) << ", "
    << "姓名: " << std::get<2>(student) << '\n';

    double gpa;
    char grade;
    std::string name;

    // 元组进行拆包
    std::tie(gpa, grade, name) = get_student(1);
    std::cout << "ID: 1, "
    << "GPA: " << gpa << ", "
    << "成绩: " << grade << ", "
    << "姓名: " << name << '\n';
}   
```

`std::get` 除了使用常量获取元组对象外，**C++14** 增加了使用类型来获取元组中的对象：

```cpp
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
std::cout << std::get<std::string>(t) << std::endl;
std::cout << std::get<double>(t) << std::endl; // 非法, 引发编译期错误
std::cout << std::get<3>(t) << std::endl;
```

#### 运行期索引

如果你仔细思考一下可能就会发现上面代码的问题，`std::get<>` 依赖一个编译期的常量，所以下面的方式是不合法的：

```cpp
int index = 1;
std::get<index>(t);
```

那么要怎么处理？

答案是，使用 `std::variant<>`（**C++ 17** 引入），提供给 `variant<>` 的类型模板参数

可以让一个 `variant<>` 从而容纳提供的几种类型的变量（在其他语言，例如 Python/JavaScript 等，表现为动态类型）：

```cpp
#include<variant>
template<size_t n, typename... T>
constexpr std::variant<T...> _tuple_index(const std::tuple<T...>& tp1, size_t i) {
    if constexpr (n >= sizeof...(T)) {
        throw std::out_of_range("越界");
    }
    if (i == n) {
        return std::variant<T...> {
            std::in_place_index<n>,
            std::get<n>(tp1)
        };
    }
    return _tuple_index<(n < sizeof...(T) - 1 ? n + 1 : 0)>(tp1,i);
}
constexpr std::variant<T...> tuple_index(const std::tuple<T...>& tpl, size_t i) {
    return _tuple_index<0>(tpl, i);
}
template <typename T0, typename... Ts>
std::ostream & operator<< (std::ostream & s, std::variant<T0, Ts...> const & v) {
    std::visit([&](auto && x) {
        s << x;
    }, v);
    return s;
}
```

这样我们就能：

```cpp
int index = 1;
std::cout << tuple_index(t, index) << std::endl;
```

#### 元组合并和遍历

还有一个常见的需求就是合并两个元组，这可以通过 `std::tuple_cat` 来实现：

```cpp
auto new_tuple = std::tuple_cat(get_student(1), std::move(t));
```

马上就能够发现，应该如何快速遍历一个元组？但是我们刚才介绍了如何在运行期通过非常数索引一个 `tuple` 那么遍历就变得简单了， 首先我们需要知道一个元组的长度，可以：

```cpp
template <typename T>
auto tuple_len(T &tp1) {
    return std::tuple_size<T>::value;
}
```

这样就能够对元组进行迭代了：

```cpp
for (int i = 0; i != tuple_len(new_tuple); ++i) {
    std::cout << tuple_index(new_tuple, i) << std::endl;
}
```

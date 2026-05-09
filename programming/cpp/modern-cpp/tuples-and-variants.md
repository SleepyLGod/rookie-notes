# Tuples and Variants

### Tuples

Looking across containers in traditional C++, except for `std::pair`, there does not seem to be a ready-made structure for storing data of different types. Usually we define our own structure for that. The limitation of `std::pair` is obvious: it can store only two elements.

#### Basic operations

There are three core functions for using tuples:

1. `std::make_tuple`: construct a tuple
2. `std::get`: get the value at a specific position in a tuple
3. `std::tie`: unpack a tuple

```cpp
#include<tuple>
#include<iostream>

auto get_student(int id) {
    // The return type is inferred as std::tuple<double, char, std::string>.
    if (id == 0) {
        return std::make_tuple(3.8, 'A', "zhangsan");
    } else if (id == 1) {
        return std::make_tuple(2.9, 'C', "Li Si");
    } else if (id == 2) {
        return std::make_tuple(1.7, 'D', "Wang Wu");
    }
    return std::make_tuple(0.0, 'D', "null");
    // If only 0 is written here, type inference fails and compilation fails.
}

int main() {
    auto student = get_student(0);
    std:cout << "ID: 0, "
    << "GPA: " << std::get<0>(student) << ", "
    << "Grade: " << std::get<1>(student) << ", "
    << "Name: " << std::get<2>(student) << '\n';

    double gpa;
    char grade;
    std::string name;

    // Unpack the tuple.
    std::tie(gpa, grade, name) = get_student(1);
    std::cout << "ID: 1, "
    << "GPA: " << gpa << ", "
    << "Grade: " << grade << ", "
    << "Name: " << name << '\n';
}   
```

Besides using a constant index to access a tuple object with `std::get`, **C++14** added support for accessing tuple elements by type:

```cpp
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
std::cout << std::get<std::string>(t) << std::endl;
std::cout << std::get<double>(t) << std::endl; // Illegal; triggers a compile-time error.
std::cout << std::get<3>(t) << std::endl;
```

#### Runtime index

If we think carefully about the code above, we can see the issue: `std::get<>` depends on a compile-time constant. Therefore, the following form is illegal:

```cpp
int index = 1;
std::get<index>(t);
```

How should this be handled?

The answer is to use `std::variant<>`, which was introduced in **C++17**. The type template parameters supplied to `variant<>` allow one `variant<>` object to hold values of several possible types. In other languages such as Python or JavaScript, this looks similar to dynamic typing:

```cpp
#include<variant>
template<size_t n, typename... T>
constexpr std::variant<T...> _tuple_index(const std::tuple<T...>& tp1, size_t i) {
    if constexpr (n >= sizeof...(T)) {
        throw std::out_of_range("out of range");
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

Then we can write:

```cpp
int index = 1;
std::cout << tuple_index(t, index) << std::endl;
```

#### Tuple concatenation and traversal

Another common requirement is merging two tuples. This can be done with `std::tuple_cat`:

```cpp
auto new_tuple = std::tuple_cat(get_student(1), std::move(t));
```

This immediately raises another question: how can a tuple be traversed quickly? We have just introduced how to index a `tuple` at runtime with a non-constant index, so traversal becomes straightforward. First, we need to know the length of a tuple:

```cpp
template <typename T>
auto tuple_len(T &tp1) {
    return std::tuple_size<T>::value;
}
```

Then the tuple can be iterated:

```cpp
for (int i = 0; i != tuple_len(new_tuple); ++i) {
    std::cout << tuple_index(new_tuple, i) << std::endl;
}
```

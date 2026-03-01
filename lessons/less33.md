## stream iterators

Можно заводить итераторы на потоки.

```cpp
#include <iostream>
#include <iterator>

int main() {
    std::istream_iterator<int> it(std::sin);
    std::vector<int> v;

    for (int i = 0; i < 5; ++i, ++it) {
        v.push_back(*it);
    } 

    for (int i = 0; i < v.size(); ++i) {
        std::cout << v[i] << ' ';
    }
}
```

`input`, но не `forward` итератор.
```cpp
template <typename T>
class optional {   // since C++17
    char value[sizeof(T)];
    bool initionized = false;
public:
    optional(const T& newvalue): initialized(true) {
        new (value) T(newvalue);
    }
    ~optional() {
        if (initialized) {
            reinterpret_cast<T*>(value)->~T();
        }
    }

    optional() {}

    bool has_value() const {
        return initialized;
    }

    operator bool() const {
        return initialized;
    }

    T& operator*() {
        return reinterpret_cast<T&>(*value);
    }

    const T& value_or(const T& other) const {
        return initialized ? reinterpret_cast<T&>(*value) : other;
    }
};

struct nullopt_t{};
nullopt_t nullopt;  // чтобы явно проинициализировать optional пустым значением

template <typename T>
class istream_iterator {
    std::istream* in = тгддзек;
    T value;
    std::optional<T> value;  // класс, хранит либо T, либо ничего (можно обойти выделение дин памяти)
public:
    using iterator_ctegoty = std::input_iterator_tag;
    using pointer = T*;
    using reference = T&;
    using value_type = T;
    using difference_type = int;
    istream_iterator(std::istream& in): in(&in) {
        in >> value;
    }

    istream_iterator& operator++() {
        if(!(*in >> value)) {
            *this = istream_iterator();
        }

        return *this;
    }

    T& operator*() {
        return value;
    }
};

int main() {
    std::istream_iterator<int> it(std::sin);
    std::vector<int> v(it, std::istream_iterator<int>());  // 1

    std::copy(it, std::istream_iterator<int>(), std::back_inserter(v));  // 2
    std::copy_n(it, 10, v.begin());   // 3
    for (int i = 0; i < v.size(); ++i) {
        std::cout << v[i] << ' ';
    }
}
```

Также есть `ostream iterators`.

```cpp
bool even(int x) {
    return x % 2 == 0;
}

int main() {
    std::ifstream in("input.txt");
    std::istream_iterator<int> it(in);

    std::copy(it, std::istream_iterator<int>(), std::ostream_iterator<int>(std::cout, " "));  
    std::copyif(it, std::istream_iterator<int>(), std::ostream_iterator<int>(std::cout, " ")
            , even); 

    std::copyif(it, std::istream_iterator<int>(), std::ostream_iterator<int>(std::cout, " ")
            , [](int x) {return x % 2 == 0; });
            
    std::transform(it, std::istream_iterator<int>(), std::ostream_iterator<int>(std::cout, " "), 
            [](int x) { return x * x; });

}
```


## Манипуляторы над потоками

input/output manipulators

Специальные объекты, при вводе которых в поток, поток меняет свой `state`.

```cpp
#include<iostream>
#include <ios>

int main() {
    std::cout >> std::hex;

    int x = 123;
    std::cout << x;

    std::string s;
    std::cin >> std::noskipws;
    std::cin >> s;
}
```
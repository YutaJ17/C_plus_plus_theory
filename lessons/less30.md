Используйте `typename std::iterator_traits<T>::value_type value`, а не просто `value_type`, если хотите получить корректный тип, так как у `T` может не быть `value_type`, а `traits` решают проблему.

### std::distance

Returns the number of hops from `first` to `last`.
`first` - iterator pointing to the first element
`end`  - iterator pointing to the end of range

Работает на `O(1)`, если через `random_access iterator`, иначе - за линию.

```cpp
#include <iterator>

template <typename Iterator>
typename std::iterator_traits<Iterator>::difference_type
distance(Iterator first, Iterator last) {
    if constexpr(std::is_same_v<
            typename std::iterator_traits<Iterator>::iterator_category,
            std::random_access_iterator_tag
            >) {
        return last - first;
    }

    int i = 0;
    for (; first != last; ++i, ++first);
    return i;
}
```

`if constexpr` дает то, что если мы подставим не итератор, то оно не начнет компилиться и не даст `CE`.

Но проверку внутри `constexpr` стоит написать вот так:


```cpp
#include <iterator>

template <typename Iterator>
typename std::iterator_traits<Iterator>::difference_type
distance(Iterator first, Iterator last) {
    if constexpr(std::is_base_of_v<
            std::random_access_iterator_tag
            typename std::iterator_traits<Iterator>::iterator_category
            >) {
        return last - first;
    }

    int i = 0;
    for (; first != last; ++i, ++first);
    return i;
}
```

Но `constexpr` появился с `C++17`. А как реализировали его до `C++17`? 
`if constexpr` - синтаксический сахар. Нужно сделать вспомогательную функцию `distance_helper`, которая третьим параметром принимает `tag`, и передать ей этим третьим параметром итератор `category`, сконструированный по умолчанию. И в зависимости от версии попадем либо в функцию с разностью, либо в функцию с циклом.

### std::advance

Продвигает итератор на `n` шагом, `n` может быть отрицательным.

### std::prev, std::next

Returns n_th predecessor of iterator it. 

---

Виды итераторов - это про то, насколько качественно заполнена информация. Наличие итераторов - достаточно существенное усложнение, например, без них `unordered_map` можно было бы реализовать гораздо проще.
Когда вы учите стихи, вы запоминаете их в одну сторону, но не в обратную. Помнить можно по-разному.


## Iterators implementation, const- and reverse- iterators



```cpp
#include <iostream>

template <typename T>
class vector {
    T* arr;
    size_t sz;
    size_t cap;

public:

    class iterator {
    private:
        T* ptr;
        iterator(T* ptr): ptr(ptr) {}

    public:
        iterator(const iterator&) = default;
        iterator& operator=(const iterator&) = default;

        T& operator*() const {
            return *ptr;
        }
        
        T* operator->() const {
            return ptr;
        }

        iterator& operator++() {
            ++ptr; 
            return *this;
        }
        
        iterator& operator++(int) {
            iterator copy = *this;
            ++ptr;
            return copy;
        }
    

        iterator begin() {
            return iterator{arr};
        }

        iterator end() {
            return iterator{arr + sz};
        }
    };

    void reserve(size_t newcap) {}

    void push_back(const T& value) {}
};
```

Есть еще `operator->*` для вызова указателей на члены класса, его тоже похорошему для итератора перегрузить.

Хотим сделать `const iterator`, это аналог указателей на константу в мире итераторов, то есть такой вид итераторов, разыменование которого будет давать `const T&`, а не `T&`. 

```cpp
class const_iterator {
private:
    const T* ptr;
    const_iterator(T* ptr): ptr(ptr) {}

    public:
    const_iterator(const const_iterator&) = default;
    const_iterator& operator=(const const_iterator&) = default;

    const T& operator*() const {
        return *ptr;
    }
        
    const T* operator->() const {
        return ptr;
    }

    const_iterator& operator++() {
        ++ptr; 
        return *this;
    }
        
    const_iterator& operator++(int) {
        const_iterator copy = *this;
        ++ptr;
        return copy;
    }
};
```

Теперь появляется естественное желание реализовать оба этих итератора, чтобы обойтись от copypaste.

Также не хватает одного важного метода. Нужно, чтобы любой итератор неявно преобразовывался в константный. Это можно реализовать через конструктор или через оператор привидения типа. Рассмотрим второй способ:

```cpp
template <bool IsConst>
class base_iterator {
public:
    using pointer_type = std::conditional_t<IsConst, const T*, T*> ptr;
    using reference_type std::conditional_t<IsConst, const T&, T&> ptr;
    using value_type = T;

private:
    pointer_type ptr;
    base_iterator(T* ptr): ptr(ptr) {}

    friend class vector<T>;   // важно добавить

public:
    base_iterator(const base_iterator&) = default;
    base_iterator& operator=(const base_iterator&) = default;

    reference_type operator*() const {
        return *ptr;
    }
        
    pointer_type operator->() const {
        return ptr;
    }

    base_iterator& operator++() {
        ++ptr; 
        return *this;
    }

    operator base_operator<true>() const {   // оператор приведения типа к base_operator<true>
        return {ptr};
    }
        
    base_iterator& operator++(int) {
        base_iterator copy = *this;
        ++ptr;
        return copy;
    }

    base_iterator begin() {
        return base_iterator{arr};
    }

    base_iterator end() {
        return base_iterator{arr + sz};
    }
};
```

Теперь `base_iterator` стоит сделать `private`, 
и внутри `vector` продолжить:
```cpp
using iterator = base_iterator<false>;
using const_iterator = base_iterator<true>;

iterator begin() {
        return {arr};
    }

iterator end() {
    return {arr + sz};
}

const_iterator begin() const {
        return {arr};
    }

const_iterator end() const {
    return {arr + sz};
}

const_iterator cbegin() const {
        return {arr};
    }

const_iterator cend() const {
    return {arr + sz};
}

operator base_operator<true>() const {   // оператор приведения типа к base_operator<true>
    return {ptr};
}
```

```cpp
int main() {
    vector<int> v;
    vector<int>::iterator it = v.begin();
    vector<int>::const_iterator cit = it;
    vector<int>::iterator it2 = cit;   // не компилируется, так и задумано
}
```

## std::reverse_iterator

Это отдельный класс стандартной библиотеки, позволяющий ходить назад, имея `bidirectional iterator`.
Это адаптер над итератором.

```cpp
template <class Iter>
class reverse_iterator;
```

Reseverses the dirirection of a given iterator, which might be atleast `bidirectional iterator`

```cpp
using reverse_iterator = std::reverse_iterator<iterator>;
using const_reverse_iterator = std::reverse_iterator<const_iterator>;
```

По аналогии, есть `crend`, `crend` и прочее.

```cpp
int main() {
    std::vector<int> v(10);
    int& x = v[5];
    v.push_back(1);
    std::cout << x;  // UB
}
```

А что если так (произойдет ли инвалидация итератора?):
```cpp
int main() {
    std::vector<int> v(10);
    std::vector<int>::iterator x = v.begin() + 5;
    v.push_back(1);
    std::cout << x;  // UB
}
```
Нельзя так делать, будет `UB`. В `deque` тоже, в отличие от ссылок и указателей.


## output_iterators

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    int a[10] = {1, 2, 3, 4, 5};
    std::vector<int> v(5);

    std::copy(v.begin(), v.end(), a);  // ok

    std::copy(a, a + 10, v.begin());  // UB
}
```
`std::copy` разыменовывает, присваивает, инкрементирует, он ничего не знает о том, что лежит под итератором.

Формально: `output iterator` - итератор, гарантирующий, что можно его разыменовывать, инкрементировать, присваивать сколько угодно раз. 
Итераторы в контейнерах такими не являются, в них нельзя было писать.

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

bool even(int x) {
    return x % 2 == 0;
}

int main() {
    int a[10] = {1, 2, 3, 4, 5};
    std::vector<int> v(5);

    std::copy_if(a, a + 10, v.begin(), &even)
}
```
Какую класс-обертку написать, чтобы это работало корректно? Нужна такая обертка над итератором вектора, которая бы предоставляла интерфейс `output` итератора. Такая обертка называется `std::back_insert_iterator`.
Это адаптер над итератором. Является `output_iterator`.

```cpp
template <class Container>
class back_insert_iterator;
```
Умеет:
- `operator=`
- `operator*`
- `operator++`

```cpp
#include <iterator>

bool even(int x) {
    return x % 2 == 0;
}

int main() {
    int a[10] = {1, 2, 3, 4, 5};
    std::vector<int> v(5);

    std::copy_if(a, a + 10,
            std::back_insert_iterator<std::vector<int>>(v), &even);
}
```

```cpp
template<Container>
class back_insert_iterator {
    Container& container;
public:
    back_insert_iterator(Container& container): container(container) {}
    back_insert_iterator& operator=(const typename Container::value_type& value) {
        container.push_back(value);
        return *this;
    }

    back_insert_iterator& operator++() {
        return *this;
    }

    back_insert_iterator& operator++(int) {
        return *this;
    }

    back_insert_iterator& operator*() {
        return *this;
    }
};

template <typename Container>
back_insert_iterator<Container> back_inserter(Container& container) {
    return {container};
}

int main() {
    int a[10] = {1, 2, 3, 4, 5};
    std::vector<int> v(5);

    std::copy_if(a, a + 10, back_inserter(v), &even);
}
```

Помимо этого есть еще `front_inserter` (все тоже самое, только наоброт), а еще есть `inserter` (принимает не только контейнер, а еще итератор в этом контейнере). `inserter` работает с любым контейнером, так как он не `push_back` делает, а вызывает метод `insert` по данному итератору у данного контейнера.

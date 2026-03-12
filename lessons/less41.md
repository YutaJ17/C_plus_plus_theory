```cpp
template <typename T>
struct allocator {
    template <typename U, typename... Args>
    void construct(U* ptr, Args&... args) {
        new (ptr) U(args...);
    }
};
```

Исправим реализацию:

```cpp
template <typename T>
struct allocator {
    template <typename U, typename... Args>
    void construct(U* ptr, Args&&... args) {
        new (ptr) U(std::forward<Args>(args)...);
    }
};
```

Реализуем `std::forward`.

### std::forward implemeptation

```cpp
template <typename T>
T&& forward(T&& value) {
    return value;
}
```
Почему так не будет работать?
Возвращаемое значение `value` имеет категорию `lvalue`, даже если его тип - `rvalue ref`.

1. 
```cpp
int x = 42;
alloc.construct(p, x);
// Args... = {int&} (lvalue ref)
// std::forward<int&>(args...) должен вернуть int&
```

2. 
```cpp
alloc.construct(p, 42);
// Args...  = {int} 
// std::forward<int>(args...) - должен вернуть int&&
```
Но функция возвращает `lvalue`, когда ожидается `rvalue` (во втором случае).


Попробуем исправить:

```cpp
template <typename T>
T&& forward(T& value) {
    return value;
}
```
Тут тоже `CE`. Если изначально принимаем `rvalue`, шаблонным параметром является `T`, и мы пытаемся вернуть `T&&`, хотя мы принимаем `T&`, то есть пытаемся инициализировать `rvalue ref` с помощью `lvalue` на возврате.


```cpp
template <typename T>
T&& forward(T& value) {
    return static_cast<T&&>(value);
}
```
Но такой `forward` ничем не отличается от `move`. 
Такая реализация корректно обрабатывает оба случая передачи.

Значит `move`, написанный ранее, неверный. 

```cpp
template <typename T>
T&& move(T& x) {
    return static_cast<T&&>(x);
}
```
`move` не должен возвращать `lvalue`. Всегда должен возвращать `rvalue`.

Правильная реализация `move`:

```cpp
template <typename T>
std::remove_reference_t<T>&& move(T&& value) {
    return static_cast<std::remove_reference_t<T>&&>(value);
}
```

Правильная реализация `forward`:

```cpp
template <typename T>
T&& forward(std::remove_reference_t<T>& value) {
    return static_cast<T&&>(value);
}
```

Почему `std::forward` должен принимать `std::remove_reference_t<T>&` вместо `T&`?
Две причины:
1. Не хочется, чтобы `forward` мог выводить тип шаблонного параметра неявно (`type deduction`). То есть теперь в `forward` нельзя не передать шаблонный параметр, будет `CE`.
2. `forward` от `rvalue` тоже может быть нужен:

```cpp
template <typename T>
T&& forward(std::remove_reference<T>& value) {
    return static_cast<T&&>(value);
}

template <typename T>
T&& forward(std::remove_reference<T>&& value) {
    static_assert(!std::is_lvalue_reference_v<T>);
    return static_cast<T&&>(value);
}
```
В теории, может быть ситуация, когда в `forward` пришло уже `rvalue` из какой-то другой функции:
```cpp
template <typename T>
struct allocator {
    template <typename U, typename... Args>
    void construct(U* ptr, Args&&... args) {
        new (ptr) U(std::forward<Args>(f<Args>(args))...);
    }
};
```
Перегрузка c rvalue форвардит `rvalue` как `rvalue`, и запрещает форвардить `rvalue` как `lvalue`.


### push_back implementation with move semantics support

```cpp
void push_back(const T& value) {
    if (sz_ == cap_) {
        reserve(cap_ > 0 ? cap_ * 2 : 1);
    }
    alloc_traits::construct(alloc_, arr_ + sz_, value);
    ++sz;
}

void push_back(T&& value) {
    if (sz_ == cap_) {
        reserve(cap_ > 0 ? cap_ * 2 : 1);
    }
    alloc_traits::construct(alloc_, arr_ + sz_, std::move(value));
    ++sz;
}
```

```cpp
template <typename... Args>
void emplace_back(Args&&... args) {
    if (sz_ == cap_) {
        reserve(cap_ > 0 ? cap_ * 2 : 1);
    }
    alloc_traits::construct(alloc_, arr_ + sz_, 
            std::forward<Args>(args)...);
    ++sz_;
}
```

Тогда в push_back можно прописать emplace_back:


```cpp
void push_back(const T& value) {
    emplace_back(value);
}

void push_back(T&& value) {
    emplace_back(std::move(value));
}
```


Осталось решить проблему реаллокации в методе reserve, чтобы избежать копирования:
```cpp
try {
    for (; index < sz_; ++index) {
        alloc_traits::construct(alloc_, newarr + index, std::move(arr_[index]));
    }
}
```
Новая проблема: move ctor может бросить исключение. Есть функция, решающая эту проблему:

```cpp
try {
    for (; index < sz_; ++index) {
        alloc_traits::construct(alloc_, newarr + index, std::move_if_noexcept(arr_[index]));
    }
}
```
Работает следующим образом: если `move ctor` не `noexcept`, то будем копировать, а не мувать.
Мораль: отмечать `move ctors` как `noexcept`, иначе объект с таким конструктором будет копироваться внутри контейнера, а не перемещаться.

Но в `push_back` решены еще не все проблемы.
```cpp
vector<string> v(5, "abc");
v.push_back(v[3]);  // UB из-за реаллокации, битая ссылка
```
Правильный `push_back` должен сначала класть элемент на ячейку n, и только потом заниматься перекладыванием предшедствующих.

---

Давайте представим, что мы отказались от принятия параметра по ссылке:

```cpp
template <typename T>
void f(T x) {

}
```
Зачем нужно 2 `push_back`, если можно принимать по значению и мувать аргумент в любом случае?
В некоторых случаях (codestyle, рекомендованный стандартом, sinceC++11), если все методы заведомо поддерживают мув семантику, то может быть не нужно делать 2 перегрузки для `lvalue` и для `rvalue`. Можно сделать 1 и по значению. То есть это будет обязанность вызывающего решить, он хочет мувнуть или скопировать. 

Тем не менее, почему `push_back` так не реализован? Для обратной совместимости, а также потому, что не все типы поддерживают мув сематику. 
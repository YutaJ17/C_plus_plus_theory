## Dependent names

```cpp
template <typename T>
struct S {
    using A = int;
};

template <>
struct S<double> {
    static const int A = 5;
};

template <typename T>
void f() {
    S<T>::A* x;
}

int main() {
    f<int>();
}
```

Почему `CE`?
`A* x` может быть как `int* x` (declaration), а может быть `5*x` (expression).
В данном примере `A` - **dependent name**(зависимое имя).
По умолчанию, все `dependent names` воспринимаются как `expressions` (по стандарту).
Чтобы оно воспринималось как имя типа, нужно прописывать `typename`:
```cpp
template <typename T>
void f() {
    typename S<T>::A* x;
}
```

---

```cpp
#include <array>
#include <iostream>

template <typename T>
struct S {
    template <int N>
    using A = std::array<int, N>;
};


template <>
struct S<double> {
    static const int A = 5;
};

template <typename T>
void f() {
    typename S<T>::A<10> x;
}

int main() {
    f<int>();
}
```
`typename S<T>::A<10> x;` здесь недостаточно слова `typename`, так как компилятор считает, что это `A` - название типа, но даже не догадывается, что это название шаблона.
Для того, чтобы это указать, нужно прописать `template`:
```cpp
template <typename T>
void f() {
    typename S<T>::template A<10> x;
}
```

---

```cpp
template<typename T>
struct S {
    template <int N>
    void foo(int) {}
};

template<>
struct S<double> {
    const int foo = 1;
};

template<typename T>
void bar(int x, int y) {
    S<T> s;
    s.foo<5>(x + y); 
}

int main() {
    bar<int>(2, 3);
    bar<double>(2, 3);
}
```
`s.foo<5>(x + y)` - шаблонная функция или expression?
Следует написать
```cpp
template<typename T>
void bar(int x, int y) {
    S<T> s;
    s.template foo<5>(x + y); 
}
```
---

```cpp
template <typename T>
struct Base {
    int x = 0;
};

template <typename T>
struct Derived: Base<T> {
    void f() {
        ++x;
    }
};

int main() {}
```

В зависимости от `T`, `x` может не быть, а может означать функцию или что-нибудь другое.
При обращении к полю шаблонного родителя, компилятор не смотрит код шаблонного родителя, чтобы понять, что это такое. Компилятор просто синтакисчески парсит.

Чтобы починить:
```cpp

template <typename T>
struct Derived: Base<T> {
    void f() {
        ++this->x;  //  или ++Base<T>::x;
    }
};
```

**Two phase translation** - две фазы генерации шаблонного кода: до подстановки `T` и после. До подстановки компилятор смотрит на синтаксис и базовые семантические проверки (имен, независимых от `T`), чтобы часть ошибок отловить на первой стадии. До подстановки `T` компилятор не в силах проверить корректность многих вещей.


## Metafunctions and type traits

Метафункции - функции от типов. 
Простейшая метафункция: проверка на равенство типов:
```cpp
template <typename T, typename U>
struct is_same {
    static constexpr bool value = false;
};

template <typename T>
struct is_same<T, T> {
    static constexpr bool value = true;
}

template <typename T, typename U>
void f(T x,U y) {
    if constexpr (is_same(T, U)::value) {  
        x = y;
    }
}

int main() {
    f<int, std::string>(5, "abc");
}
```
`if constexpr` для того, чтобы `if` вычислялся в `CompileTime`. Без `constexpr` было бы `CE`. 
`constexpr` говорит комлилятору выкинуть `if statement` из кода, если оно ложно, и не смотреть на то, что внутри него. Но условие должно быть в `Compile Time` проверяемо.


Бывают такие метафункции, которые по типу возвращают другой тип:
```cpp
template <typename T>
struct remove_reference {
    using type = T;
};

template <typename T>
struct remove_reference<T&> {
    using type  = T:
}

template <typename T>
void f() {
    typename remove_reference<T>::type x;  // typename из-за dependent names
}

int main() {}
```

---

```cpp
template <typename T>
struct remove_const {
    using type = T;
};

template <typename T>
struct remove_const<const T> {
    using type = T:
}
```

Такие метафункции есть в заголовочном файле `<type_traits>`.

```cpp
template <bool B, typename T, typename F>
struct conditional {
    using type = F;
};

template <typename T, typename F>
struct conditional<true, T, F> {
    using type = T;
};

template <typename T>
void f() {
    typename remove_reference<T>::type x;
}
```
Это тернарный метаоператор, то есть тернарный оператор для типов.

Есть `type_traits`, есть шаблонные `using`. В первых неудобно каждый раз прописывать `typename`. Почему бы не доопределить в `stl` шаблонные `using` для всех `type_traits`:
```cpp
template <bool B, typename T, typename F>
using conditional_t = typename conditional<B, T, F>::type;
```

Мораль: не используйте структуры выше в чистом виде, используйте `alias`. 
Если нужно использовать, например, `remove_reference`, не нужно писать 
`typename std::remove_reference<T>::type`. Начиная с `C++14` вы можете писать
`remove_reference_t<T>`. Это распростроняется на все `type_traits`, которые возвращают тип.
Заодно в `C++14` добавили шаблонные переменные.
В `C++17` придумали доопределить `type_traits`, возвращающие `value`:
```cpp
template <typename T, typename U>
const bool is_same_v = is_same<T, U>::value;
```


## Variadic templates (since C++)

Шаблоны с переменным количеством аргументом.

```cpp
template <typename... Types>
void f(Types... tx) {

}
```
`Types` - особый вид шаблонного аргумента, который представляет собой пакет типов.
Этот пакет можно распаковать. Для этого нужно написать `Types...`.

В первой строке `template <typename... Types>` многоточие означает объявление пачки типов,
а во второй строке `Types... tx` - распаковку.

`tx` - пачка переменных, ее можно куда-то передать дальше: `g(tx...)`. Пакет может быть пустым.

```cpp
void print() {}

template <typename Head, typename... Tail>
void print(const Head& head, const Tail&... tail) {
    std::cout << head << ' ';
    print(tail...);
}

int main() {
    print(1, 2.0, "abc");
}
```

---

```cpp
#include <iostream>
#include <type_traits>

template <typename First, typename Second, typename... Types>
struct is_homogeneous {
    static constexpr bool value = std::is_same_v<First, Second>
            && is_homogeneous<Second, Types...>::value;
};

template <typename First, typename Second>
struct is_homogeneous<First, Second> {
    static constexpr bool value = std::is_same_v<First, Second>;
};
```

У метапеременных бинарные операторы работают также лениво (правая часть не вычисляется, если левая - ложна). Но "_не вычисляется_" не значит "_не истанцируется_". Шаблонная подстановка все равно происходит. 

Есть оператор `sizeof...(tail)`, где `tail` - пакет. Он в `CompileTime` возвращает число, равное размеру пакета. 

Общий принцип работы с `variadic templates` - откусывать `head` и шаблонной рекурсией постепенно обрабатывать `tail`.
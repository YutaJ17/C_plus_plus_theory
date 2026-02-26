## Fold expressions (C++17)

```cpp
template <typename... Types>
struct all_pointers {
    static const bool value = (std::is_pointerv<Types> && ...);
};
```

Пакет можно засунуть в *fold expression*. Означает: повторить для всех типов из пакета.


```cpp
template <typename... Types>
struct all_pointers {
    static const bool value = (std::is_pointerv<Types> && ...);
};

template <typename Head, typename... Tail>
struct is_homogeneous {
    static const bool value = (std::is_same_v<Head, Tail> && ...);
};

void print() {}

template <typename... Types>
void print(const Types&... types) {
    (std::cout << ... << types);
}
```

4 типа **fold expressions**:
1. (pack op ...)  - unary right fold

2. (... op pack)  - unary left fold
3. (pack op ... op init)  - binary right fold
4. (init op ... op pack)  - binary left fold

pack - пакет, op - бинарный оператор, init - инициализатор.

**op**	-   any of the following 32 binary operators: `+ - * / % ^ & | = < > << >> += -= *= /= %= ^= &= |= <<= >>= == != <= >= && || , .* ->*`. In a binary fold, both ops must be the same.

**pack**	-	an expression that contains an unexpanded pack and does not contain an operator with precedence lower than cast at the top level (formally, a cast-expression)

**init**	-	an expression that does not contain an unexpanded pack and does not contain an operator with precedence lower than cast at the top level (formally, a cast-expression)

Note that the opening and closing parentheses are a required part of the fold expression.


The instantiation of a fold expression expands the expression e as follows:

1) Unary right fold (`E` op ...) becomes (`E_1` op (... op (`E_N-1` op `E_N`)))
2) Unary left fold (... op `E`) becomes (((`E_1` op `E_2`) op ...) op `E_N`)
3) Binary right fold (`E` op ... op `I`) becomes (E1 op (... op (`E_N−1` op (`E_N` op `I`))))
4) Binary left fold (`I` op ... op `E`) becomes ((((`I` op `E_1`) op `E_2`) op ...) op `E_N`)
(where N is the number of elements in the pack expansion)


```cpp
template<typename... Args>
void printer(Args&&... args) {
    (std::cout << ... << args) << std::endl;
}
```


## CRTP

**Curiously Recursive Template Pattern**

```cpp
template <class T>
struct Base {
    void interface() {
        // ...
        static_cast<T*>(this)->implementation();
    }

    static void static_func() {
        // ...
        T::static_sub_func();
        // ...
    }
};

struct Derived : Base<Derived> {
    void implementation();
    static void static_sub_func();
};

int main() {}
```
Отдаленно может напоминать механизм виртуальных функций.

Как это может компилироваться?





Вот так бы не скомпилировалось:
```cpp

template <class T>
struct Base {
    T x;  // из-за объекта типа T не компилируется
    T* ptr;  // это мешать не будет
    T& ref;  // это тоже ok, компилируется

    void interface() {
        // ...
        static_cast<T*>(this)->implementation();
    }

    static void static_func() {
        // ...
        T::static_sub_func();
        // ...
    }
};

struct Derived : Base<Derived> {
    void implementation();
    static void static_sub_func();
};

int main() {}
```

Чтобы объявить `T*` или `T&` как поле, не нужно знать определение `Derived`, только его объявление.
Если создать объект типа `T`, до будет зацикленность при компиляции -> CE. 


## Expression templates

Представьте, что хочется реализовать класс `vector` (в математическом смысле, как упорядоченный набор чисел).
Хочется уметь складывать векторы покомпонентно. А также брать `[]` от суммы. Иногда нужно подсчитать `i` элемент суммы, не считая всю сумму. Ленивое вычисление засчет шаблонов.

Происходит имитация синтаксического дерева разбора. Есть 2 типа `expressions`: `leaf`(лист) и `not leaf`.

Здесь используется `CRTP`. 

```cpp
template <typename E>
class VecExpression {
public:
    static constexpr bool is_leaf = false;

    [[nodiscard]]
    double operator[](size_t i) const {
        // Delegation to the actual expression type. This avoids dynamic polymorphism (a.k.a. virtual functions in C++)
        return static_cast<E const&>(*this)[i];
    }

    [[nodiscard]]
    size_t size() const { 
        return static_cast<E const&>(*this).size(); 
    }
};


class Vec3 : public VecExpression<Vec3> {
private:
    array<double, 3> elems;
public:
    static constexpr bool is_leaf = true;

    [[nodiscard]]
    double operator[](size_t i) const { 
        return elems[i]; 
    }

    double& operator[](size_t i) { 
        return elems[i]; 
    }

    [[nodiscard]]
    size_t size() const { 
        return elems.size(); 
    }

    // construct Vec using initializer list 
    Vec3(initializer_list<double> init) {
        std::ranges::copy(init, elems.begin());
    }

    // A Vec can be constructed from any VecExpression, forcing its evaluation.
    template <typename E>
    Vec3(VecExpression<E> const& expr) {
        for (size_t i = 0; i != expr.size(); ++i) {
            elems[i] = expr[i];
        }
    }
};



template <typename E1, typename E2>
class Vec3Sum : public VecExpression<Vec3Sum<E1, E2>> {
private:
    // cref if leaf, copy otherwise
    typename conditional<E1::is_leaf, const E1&, const E1> _u;
    typename conditional<E2::is_leaf, const E2&, const E2> _v;
public:
    static constexpr bool is_leaf = false;

    Vec3Sum(E1 const& u, E2 const& v): _u(u), _v(v) {
        assert(u.size() == v.size());
    }

    double operator[](size_t i) const { return u[i] + v[i]; }

    size_t size() const { return v.size(); }
};
  
template <typename E1, typename E2>
Vec3Sum<E1, E2> operator+(VecExpression<E1> const& u, VecExpression<E2> const& v) {
   return Vec3Sum<E1, E2>(*static_cast<const E1*>(&u), *static_cast<const E2*>(&v));
}



int main() {
    Vec3 v0{23.4,  12.5,  144.56};
    Vec3 v1{67.12, 34.8,  90.34};
    Vec3 v2{34.90, 111.9, 45.12};
    
    // Following assignment will call the ctor of Vec3 which accept type of 
    // `VecExpression<E> const&`. Then expand the loop body to 
    // a.elems[i] + b.elems[i] + c.elems[i]
    Vec3 sumOfVecType = v0 + v1 + v2; 

    for (size_t i = 0; i < sumOfVecType.size(); ++i) {
        std::println("{}", sumOfVecType[i]);
    }

    // To avoid creating any extra storage, other than v0, v1, v2
    // one can do the following (Tested with C++11 on GCC 5.3.0)
    auto sum = v0 + v1 + v2;
    for (size_t i = 0; i < sum.size(); ++i) {
        std::println("{}", sum[i]);
    }
    // Observe that in this case typeid(sum) will be Vec3Sum<Vec3Sum<Vec3, Vec3>, Vec3>
    // and this chaining of operations can go on.
}
```

---

## Exceptions

Исключения.

### Idea of exceptions and basic examples

В чем глобальная проблема? Какие-то функции могут вызываться неудачно, и мы хотим это понимать. При этом у функции по возвращаемому параметру можно не понять, что что-то не так. Один из самых популярных путей - возвращать код ошибки. А если мы хотим, чтобы функция возвращала что-то полезное, помимо кода возврата?
Раньше приходилось подавать в функцию указатель на то, куда мы хотим положить результат, и чтобы функция возвращала код ошибки.

```cpp
#include <cstdlib>

int main() {
    void p = malloc(1234567);  // can return valid ptr or nullptr
}
```

Аналогично работала функция `atoi`. Если забыть проверить, что возвращаемое значение неравно 0, то будут проблемы. Ранее обнаружение ошибок - залог хорошего кода.

В `C++` появился механизм исключений.
```cpp
#include <exception>

int divide(int a, int b) {
    if (b == 0) {
        throw std::logic_error("Divide by zero!");  // исключение
    }
    return a / b;
}

int main() {
    divide(1, 0);
}
```

При выходе из функции через `throw` происходит раскрутка стека `main -> f1 -> f2 -> ... -> f10 (error)`. (Раскрутка в обратную сторону). Чтобы обработать эту ошибку, есть синтаксическая конструкция `try catch`.

```cpp
#include <exception>

int divide(int a, int b) {
    if (b == 0) {
        throw std::logic_error("Divide by zero!");
    }
    return a / b;
}

int main() {
    try {
        divide(1, 0);
    } catch (std::logic_error& err) {  
        std::cout << err.what() << std::endl;  
    }
}
```

`std::logic_error&` по ссылке, так как если поймать исключение по значению, будет создана лишняя копия.

`try catch` - неделимая конструкция.

Это дает возможность писать более лаконичный код.

Но `try catch` - дорогое удовольствие, даже тяжелее, чем `new delete`.

```cpp
int main() {
    try {
        new int[400'000'000'000'000];
    } catch (std::logic_error& err) {
        std::cout << err.what();    // std::bad_alloc
    }
}
```

Исключения происходят в `RunTime`.


## Difference between exceptions and runtime errors

```cpp
int main() {
    std::vector<int> v;
    v[1000000] = 1;    // Segmentation fault
}
```

```cpp
int main() {
    std::vector<int> v;
    try {
        v[1000000] = 1;
    } catch (...) {
        std::cout << "caught" << std::endl;  // ничего не поймает, ведь SegFault - не exception
    }
}
```

```cpp
int main() {
    std::vector<int> v;
    int x;
    std::cin >> x;  // 0
    try {
        std::cout << 5 / x;
    } catch (...) {
        std::cout << "caught" << std::endl;  // также ничего не поймаем
    }
}
```
Выведет `floating point exception: core dumped`. Но это не `expection` уровня плюсов, а `exception` другого уровня - уровня процессора.
`SegFault`, `floating point exc`, `Aborted` - 3 вида `RunTimeError`. Это исключения с точки зрения процессора.
А исключения, которые мы кидаем и ловим - это более высокий уровень абстракции, разные уровни ошибок.
`Exceptions` уровня плюсов можно поймать, а если не смочь, то мы упадем с вердиктом `Aborted`, а `exceptions` уровня процессора поймать нельзя. 
Пример: процессор ловит деление на 0, зовет операционку, которая убивает программу, которая это совершила. 

Еще есть класс стандартной библиотеки `std::runtime_error`, который является частным случаем `std::exception`.


### std::exception
Member functions:
- `constructor`
constructs the exception object
(public member function)

- `destructor` (virtual)
destroys the exception object
(virtual public member function)

- `operator=`
copies exception object
(public member function)

- `what` (virtual)
returns an explanatory string
(virtual public member function)

### std::bad_typeid

```cpp
class bad_typeid : public std::exception
```

Member functions:
- `constructor`
constructs a new bad_typeid object
(public member function)

- `operator=`
replaces the bad_typeid object
(public member function)

- `what`
returns the explanatory string
(public member function)


### Standard exceptions

В чем отличие `logic error` и `runtime error`? В первой ошибке виноват пользователь, а во второй что-то пошло не так.

- `logic_error`
- `invalid_argument`
- `domain_error`
- `length_error`
- `out_of_range`
- `future_error` (since C++11)
- `runtime_error`
- `range_error`
- `overflow_error`
- `underflow_error`
- `regex_error`
- `system_error`
- `ios_base::failure` (since C++11)
- `filesystem::filesystem_error` (since C++17)
- `tx_exception` (TM TS)
- `nonexistent_local_time`
- `ambiguous_local_time`
- `format_error` (since C++20)
- `bad_typeid`
- `bad_cast`
- `bad_any_cast`
- `bad_optional_access` (since C++17)
- `bad_expected_access` (since C++23)
- `bad_weak_ptr`
- `bad_function_call` (since C++11)
- `bad_alloc`
- `bad_array_new_length` (since C++11)
- `bad_exception`
- `bad_variant_access` (since C++17)

## Специализация шаблонных функций

```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

int main() {
    f(0, 0);  // 2
}
```

```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <>
void f(int, int) {
    std::cout << 3;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

int main() {
    f(0, 0);  // 2
}
```

```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, int) {
    std::cout << 3;
}

int main() {
    f(0, 0);  // 3
}
```

Перегрузка функций и специализация шаблонных функций - следует различать. 

```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, int) {
    std::cout << 3;
}

void f(int, int) {
    std::cout << 4;
}

int main() {
    f(0, 0);  // 4
}
```
Итеративный процесс:
1. Компилятором составляется полный список функций, которые можно вызвать (и шаблонные, и нешаблонные функции)
2. Для каждого найденного шаблона компилятор пытается подставить типы из аргументов вызова и понять, получается ли валидный код
3. Выбор наиболее специализированного кандидата: если есть нешаблонная, идеально подходящая функция, то выбирается она. Если такой нет, то выбирается наиболее специализированный основной шаблон.

   

Частичная специализация запрещена для функций стандартом (и это логично, для такого существует перегрузка).



```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, double) {
    std::cout << 3;
}

int main() {
    f(0, 0);  // 2
}
```


```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, double) {
    std::cout << 3;
}

void f(int, long long) {
    std::cout << 4;
}

int main() {
    f(0, 0);  // 2
}
```


```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, double) {
    std::cout << 3;
}

void f(int, long long) {
    std::cout << 4;
}

int main() {
    f(0, 0.0);  // 3
}
```

```cpp
#include <iostream>

template <typename T, typename U>
void f(T, U) {
    std::cout << 1;
}

template <typename T>
void f(T, T) {
    std::cout << 2;
}

template <>
void f(int, double) {
    std::cout << 3;
}

void f(int, long long) {
    std::cout << 4;
}

int main() {
    f(0, 0.0f);  // 1
}
```

Версия шаблона выбирается по принципу: более частный из подходящих.

## NTTP

Non-type template parameters.
Не только типы могут быть параметром шаблона. Ими могут быть, например, числа:

```cpp
#include <iostream>
#include <array>

template <typename W, size_t N>
class array {
    T arr[N];
}; 

template <size_t M, size_t N, typename Field = double>
class Matrix {};

template <size_t N, typename Field = double>
using SquareMatrix = Matrix<N, N, Field>;

template <size_t M, size_t K, size_t N, typename Field>
Matrix<M, N, Field> operator*(const Matrix<M, K, Field& a>, const Matrix<K, N, Field>& b);

int main()
    std::array<int, 100> a;

    Matrix<5, 5> m;
    SquareMatrix<5> sm;

    int x = 5;
    const y = 10;
    Matrix<x, x> m1; // x - lvalue, будет CE
    Matrix<y, y> m2; // а так можно, но не всегда
```

`constexpr` (C++11) для означает, что мы хотим, чтобы значение константной переменной было известно на момент компиляции.

```cpp
int main() {
    int x;
    std::cout << x;
    constexpr int y = x;  // будет ошибка благодаря constexpr
}
```

## Template template parameters

```cpp
template <typename T, template <typename> class Container = std::vector>
class Stack {
    Container<T> container;
};

int main() {
    Stack<int, std::vector> s;  // CE
}
```
Почему `CE`? Ошибка в том, что у вектора (как и у всех stl) вторым шаблонным параметром, по умолчанию, идет `std::allocator`.
Нужно вот так:
```cpp
template <typename T, template <typename, typename> class Container = std::vector>
class Stack {
    Container<T, std::allocator<T>> container;
};

int main() {
    Stack<int, std::vector> s;  
}
```

## Basic compile time computations

```cpp
template <int N>
struct Fibonacci {
    static const int value = Fibonacci<N-1>::value + FIbonacci<N-2>::value;
};

template <>
struct Fibonacci<1> {
    static constexpr int value = 1;
};

template <>
struct FIbonacci<0> {
    static const int value = 0;
};


int main() {
    std::cout << Fibonacci<20>::value;
}
```
У компилятора есть встроенный лимит на глубину рекурсивного инстанцирования шаблона.
Код выше работает за линейную асимптотику.

```cpp
template <int N, int D>
struct IsPrimeHelper {
    static constexpr bool value = N & D == 0 ? false : IsPrimeHelper<N, D - 1>::value;
};

template <int N>
struct IsPrimeHelper<N, 1> {
    static const bool value = true;
}

template <int N>
struct IsPrime {
    static constexpr bool value = IsPrimeHelper<N, N-1>::value;
};

template <>
struct IsPrime<1> {
    static constexpr bool value = false;
};

template <int N>
const bool is_prime = Isprime<N>::value;

int main() {
    static_assert(is_prime<257>);
    std::cout << IsPrime<257>;  // мета функция
}
```


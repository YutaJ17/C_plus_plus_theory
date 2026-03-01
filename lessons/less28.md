## Exception safety

4 уровня безопасности относительно исключений:
- `nothrow exception guarantee` - не кидает исключения
- `strong exception guarantee` - функция при броске исключения откатывает изменения назад
- `basic exception guarantee` - по крайней мере все останется в валидном состаянии, нет `UB`, нет `memory lick`, инварианты не нарушены
- `no exception guatantee` - нет гарантии безопасности


Функцию можно пометить как некидающую исключения:
```cpp
void f() noexcept {}
```
`noexcept` - exception specification (спецификатор)

```cpp
void f() noexcept try {
    // ...
} catch {
    // ...
}
```

Если из `noexcept function` вылетит исключение, то будет `std::terminate`.

```cpp
template <typename T>
void f() noexcept(std::id_reference_v<T>) {}
```

Условный `noexcept` - проверка в `CompileTime`.


```cpp
template <typename T>
void g() {}

template <typename T>
void f() noexcept {}

int main() {
    std::cout << noexcept(g<int>());  // 0
}
```

operator `noexcept` в `CompileTime` проверяет, является ли выражение под ним `noexcept`. Возвращает `bool`.Если выражение под ним - вызов функции, то проверяет, помечен ли вызов функции как `noexcept`. А если выражение под ним состоит из вызовов каких-то стандартных операторов, то `noexcept` являются все операторы, кроме `new`, `dynamic_cast`, `typeid`, `throw`. 


```cpp
template <typename T>
void g() {}

template <typename T>
void f() noexcept(noexcept(g<T>())) {}  // noexcept в зависимости от условия, была ли noexcept функция g
```
Теперь `f()` - `noexcept` тогда и только тогда, когда функция `g()` тоже `noexcept`.


`std::vector<T, allocator>`
```cpp
vector() noexcept(noexcept(Allocator()));
```

Оператор `at()` у вектора кидает исключение при выходе за границу.

А оператор `operator[]` не является `noexcept`, хоть и не кидает исключений. Почему? Семантика слова `noexcept` не совсем в том, что вы обещаете не кидать исключения. Этим словом помечаются те функции, которые в принципе не могут пойти неудачно. 

Деструкторы `noexcept` по умолчанию, начиная с `C++`. 


## Containers and iterators

### std::vector implementation

```cpp
#include <iostream>

template <typename T>
class vector {
    T* arr;
    size_t sz;
    size_t cap;

public:
    void reserve(size_t newcap) {
        T* newarr = new T[newcap];
    }

    void push_back(const T& value) {
        if (sz == cap) {
            reserve((cap > 0) ? cap * 2 : 1);
        }
    }
};

struct S {
    int x;
    S(int x): x(x) {}
};

int main() {
    int x;
    S s(x);
    vector<S> v;
    v.push_back(S(1));
}
```

Нужно научится делать реаллокацию, не создавая при этом объекты. Делая `reserve`, мы хотим выделить память, а объекты будем класть по мере надобности.


```cpp
#include <iostream>

template <typename T>
class vector {
    T* arr;
    size_t sz;
    size_t cap;

public:
    void reserve(size_t newcap) {
        T* newarr = reinterpret_cast<T*>(new char[newcap * sizeof(T)]);   // 1 грабли
        for (size_t index = 0; index < sz; ++index) {
            newarr[index] = arr[index];   // 2 грабли (UB)
        }
    }

    void push_back(const T& value) {
        if (sz == cap) {
            reserve((cap > 0) ? cap * 2 : 1);
        }
    }
};

struct S {
    int x;
    S(int x): x(x) {}
};

int main() {
    int x;
    S s(x);
    vector<S> v;
    v.push_back(S(1));
}
```

Важно понять, почему в `newarr[index] = arr[index];` происходит `UB`. Потому, что память под `newarr` выделена как `char[]`, а не как массив объектов типа `T`. Присваивание `newarr[index] = arr[index];` пытается обратиться к объекту, который еще не был создан - в памяти лежат сырые байты, а объекты типа T.

```cpp
struct Strange {  // такой тип не переживет реаллокацию через memcpy
    int x;
    int& r;
    Strange(int y): x(y), r(x) {}
};
```

По сути, чтобы решить проблему, нужно по данному адресу вызвать конструктор сырого типа.
Для этого нужна особая форма оператора `new`, а именно - `placement new`.

```cpp
void reserve(size_t newcap) {
    if (newcap <= cap) {
        return;
    }

    T* newarr = reinterpret_cast<T*>(new char[newcap * sizeof(T)]); 
    for (size_t index = 0; index < sz; ++index) {
        new(newarr + index) T(arr[index]);  // placement new
    }

    for (size_t index = 0; index < sz; ++index) {
        // нужен явный вызов деструктора объекта 
        (arr + index)->~T();
    }
    delete[] reinterpret_cast<char*>(arr);

    arr = newarr;
    cap = newcap;
}
```

`new` может кидать исключения, нужно подумать про `exception safety`.

```cpp
T* newarr = reinterpret_cast<T*>(new char[newcap * sizeof(T)]); 
```
Тут вы еще ничего не успели испортить.

```cpp
new(newarr + index) T(arr[index]);  // а вот тут уже успели
```



```cpp
void reserve(size_t newcap) {
    if (newcap <= cap) {
        return;
    }

    T* newarr = reinterpret_cast<T*>(new char[newcap * sizeof(T)]); 
    
    size_t index = 0;
    try {
        for (; index < sz; ++index) {
            new(newarr + index) T(arr[index]);  // placement new
        }
    } catch (...) {
        for (size_t oldindex = 0; oldindex < index; ++index) {
            (newarr + oldindex)->~T();
        }
        delete[] reinterpret_cast<char*>(arr);
        throw;
    }

    for (size_t index = 0; index < sz; ++index) {
        (arr + index)->~T();
    }
    delete[] reinterpret_cast<char*>(arr);

    arr = newarr;
    cap = newcap;
}
```

`push_back` выглядит примерно также, только нужно еще обработать ситуацию, если мы не смогли положить последний элемент.


## vector<bool>

```cpp
template <typename T>
class vector {};

template <>
class vector<bool> {
    char* arr;
    size_t sz;
    size_t cap;

public:
//
}

int main() {
    std::vector<bool> v(10);
    v[5] = true;
}
```

Обращение `[]` к `vector<bool>` дает `bit reference`.


```cpp
template <typename T>
class vector {};

template <>
class vector<bool> {
    char* arr;
    size_t sz;
    size_t cap;
    
    struct BitReference {
        char* cell;
        uint8_t = index;

        BitReference(char* cell, uint8_t index) {}

        void operator=(bool b) {
            if (b) {
                *cell |= (1 << index);
            } else {
                *cell &= ~(1 << index);
            }
        }

        operator bool() const {
            return *cell & (1 << index);
        }
    };
    

public:
    BitReference operator[](size_t index) {
        return BitReference{arr + index / 8, index % 8};
    }
}

int main() {
    std::vector<bool> v(10);
    v[5] = true;   // присваиваем rvalue
}
```
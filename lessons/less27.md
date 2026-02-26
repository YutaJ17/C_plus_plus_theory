Мы знаем 3 низкоуровневые вида `RunTimeError`:
- `SegFault`
- `FPE`
- `Aborted`

Есть и другие, например, нажать `ctrl+C` на клавиатуре.

Какие есть высокоуровневые причины, приводящие к ошибкам выше?
- к `SegFault`: `array out of range`, `nullptr reference`, `stack overflow`
- к `FPE`: `/0`
- к `Aborted`: `std::abort()` <-- `std::terminate()` <-- (`uncaught exception` или `pure virtual function call` и прочее)
  
к функции `std::abort()` также приводит `false assertion`.

`C++ exceptions` - это только `uncaught exceptions`.


## Exception handling

```cpp
#include <iostream>

struct A {
    A() { std::cout << "A" << std::endl; }
    ~A() { std::cout << "~A" << std::endl; }
    A(const A&) { std::cout << "copy" << std::endl; }
};

void f(int x) {
    A a;
    if (x == 0) {
        throw a;
    }
}

int main() {
    try {
        f(0);
    } catch (...) {
        std::cout << "caught" << std::endl;
    }
}
```
Во время `throw a` произойдет копирование A (так как мы используем copy constructor, а не move constructor, но об этом позже).

В какой памяти будет хранится исключение, которое мы бросаем? В динамической памяти (логично, что не на стеке, иначе бы вышли на `scope`).

1. создаем A
2. создаем копию A 
3. уничтожаем A
4. caught
5. уничтожаем копию A


А если написать так:

```cpp
int main() {
    try {
        f(0);
    } catch (A a) {
        std::cout << "caught" << std::endl;
    }
}
```
то пришлось бы еще раз скопировать A (из динамической памяти на стек).


```cpp
int main() {
    try {
        f(0);
    } catch (A& a) {
        std::cout << "caught" << std::endl;
    }
}
```
А тут второй копии не будет, так как принимаем по ссылке.
На бросание исключений еще выделяем динамическую память. Бросание исключений - это очень дорогая операция. 

А если не получится выделить динамическую память?

```cpp
#include <iostream>

struct A {
    A() { std::cout << "A" << std::endl; }
    ~A() { std::cout << "~A" << std::endl; }
    A(const A&) { std::cout << "copy" << std::endl; }
};


void f(int x) {
    A a;
    if (x == 0) {
        throw a;
    }
}

int main() {
    try {
        new int[400'000'000'000];
    } catch (A& a) {
        std::cout << "caught" << std::endl;
    } catch (std::bad_alloc& ex) {
        std::cout << &ex << std::endl;
    }
}
```
В какой памяти будет храниться `bad_alloc`? На случай, если `new` кинет исключение, в статической памяти зарезервировано место, куда положить `bad_alloc` (emergency buffer).

Внутри `catch` можно прописать `throw`.
```cpp
int main() {
    try {
        new int[400'000'000'000];
    } catch (A& a) {
        std::cout << "caught" << std::endl;
        throw;   // без аргумента: пускаем дальше лететь исключение, которое летело
    } catch (std::bad_alloc& ex) {
        std::cout << &ex << std::endl;
    }
}
```

А можно написать 
```cpp
int main() {
    try {
        new int[400'000'000'000];
    } catch (A& a) {
        std::cout << "caught" << std::endl;
        throw a;    // старое a уничтожается, дальше летит новое a
    } catch (std::bad_alloc& ex) {
        std::cout << &ex << std::endl;
    }
}
```


```cpp
#include <iostream>

struct A {
    A() { std::cout << "A" << std::endl; }
    ~A() { std::cout << "~A" << std::endl; }
    A(const A&) { std::cout << "copy" << std::endl; }
};


void f(int x) {
    A a;
    if (x == 0) {
        throw a;
    }
}

int main() {
    try {
        try {
            f(0);
        } catch (A& a) {
            std::cout << "caught" << a& << std::endl;
            throw;
        }
    } catch (A& a) {
        std::cout << "caught again" << a& << std::endl;
    }
}
```

Если изнутри `catch` делать `throw`, и в этом же блоке были другие `catch`, то они проигнорируются.

Если исключение не ловится, то компилятор не дает гарантии, что вызовется деструктор и другие специальные методы. Это `UB`.


## Multiple catch

```cpp
int main() {
    try {
        throw 1;
    } catch (double d) {
        std::cout << "double";
    } catch (long long l) {
        std::cout << "long long";
    } catch (...) {
        std::cout << "other";
    }
}
```
Выведется other.

```cpp
int main() {
    try {
        throw 1;
    } catch (double d) {
        std::cout << "double";
    } catch (long long l) {
        std::cout << "long long";
    } 
}
```
`terminated`. В первом примере вывелся `other` не потому, что он лучше всего подходит, а потому, что он единственный, кто подходит. Правило перегрузки на `catch` не распростроняется.
Но есть исключения:
1. конверсия в const
2. можно ловить по ссылке на родителя то, что было брошено как наследник

На сами `catch` перегрузка также не распростроняется, какой первый подошел, тот и срабатывает:
```cpp
#include <iostream>

struct Mom {};
struct Son : Mom {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Mom) {
        std::cout << "caught Mom";
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Вывод: `caught Mom`. 


```cpp
#include <iostream>

struct Mom {};
struct Son : Mom {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Mom) {
        std::cout << "caught Mom";
        throw;                     // terminate, так как catch на том же уровне не рассматриваются
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```


```cpp
#include <iostream>

struct Mom {};
struct Son : private Mom {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Mom) {
        std::cout << "caught Mom";    
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Вывод: `caught Son`, так как наследование приватное, `main()` не видит, что `Son` наследуется от `Mom`.



```cpp
#include <iostream>

struct Mom {};
struct Son : private Mom {
    friend int main();
};

int main() {
    try {
        Son s;
        throw s;
    } catch (Mom) {
        std::cout << "caught Mom";    
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Любопытно, но выводит `caught Son`, а не `caught Mom`. 


```cpp
#include <iostream>

struct Granny {};
struct Dad: Granny {};
struct Mom: Granny {};
struct Son :  Mom, Dad {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Granny) {
        std::cout << "caught Granny";    
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Вывод: `caught Son`.


```cpp
#include <iostream>

struct Granny {};
struct Dad: Granny {};
struct Mom: private Granny {};
struct Son : private Mom, Dad {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Granny) {
        std::cout << "caught Granny";    
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Вывод: `caught Son`.

```cpp
#include <iostream>

struct Granny {};
struct Dad: virtual Granny {};
struct Mom: virtual Granny {};
struct Son : Mom, Dad {};

int main() {
    try {
        Son s;
        throw s;
    } catch (Granny) {
        std::cout << "caught Granny";    
    } catch (Son) {
        std::cout << "caught Son";
    } catch (...) {
        std::cout << "other";
    }
}
```
Вывод: `caught Granny`.


## RAII

Идиома.
Resource Aquizition Is Initialization.
Захват(приобритение) ресурса есть инициализация.
```cpp
void g(int y) {
    if (y == 0) {
        throw 1;
    }
}

void f(int x) {
    int* p = new int(x);
    g(*p);
    delete p;
}

int main() {
    f(5);
}
```
Проблема в том, что между `new` и `delete` происходит `throw` и `delete` никто не вызывает. Есть мнение, что исключение не стоит использовать вообще. Но есть и другое мнение, что эту проблему можно решить при помощи ООП.


```cpp
struct Wrapper {
    int* p;
    Wrapper(int* p): p(p) {}
    ~Wrapper() { delete p; }
};

void g(int y) {
    if (y == 0) {
        throw 1;
    }
}

void f(int x) {
    int* p = new int(x);
    g(*p);
}

int main() {
    f(5);
}
```

При выделении ресурса вы создаете обертку, которая этот ресурс освобождает в деструкторе. На самом деле такая обертка называется _умным указателем_. Правда, у такого указателя есть своя проблема: что если этот умный указателей хочется куда-то отдать?


```cpp
struct SmartPtr {
    int* p;
    SmartPtr(int* p): p(p) {}
    ~SmartPtr() { delete p; }
    int& operator*() {
        return *p;
    }
};

void g(SmartPtr p) {
    if (*p == 0) {
        throw 1;
    }
}

void f(int x) {
    SmartPtr w(new int(x));
    g(p);
}

int main() {
    f(5);
}
```
Если передавать SmartPtr по значению, то все крашнется: создастся копия, а поскольку `copy constructor` не задан, то компилятор напишет тривиальный. В итоге два раза сработает `destructor` одного и того же объекта.
Такой SmartPtr нельзя копировать. Но в таком случае не будет выполнено `Rule of three`.


```cpp
struct SmartPtr {
    int* p;
    SmartPtr(int* p): p(p) {}
    SmartPtr(const SmartPtr&) = delete;
    SmartPtr& operator=(const SmartPtr&) = delete;
    ~SmartPtr() { delete p; }
    int& operator*() {
        return *p;
    }
};

void g(SmartPtr& p) {
    if (*p == 0) {
        throw 1;
    }
}

void f(int x) {
    SmartPtr w(new int(x));
    g(p);
}

int main() {
    f(5);
}
```

Хотелось бы, чтобы этот SmartPtr был не только для int:


```cpp
template <typename T>
struct SmartPtr {
    T* p;
    SmartPtr(T* p): p(p) {}
    SmartPtr(const SmartPtr&) = delete;
    SmartPtr& operator=(const SmartPtr&) = delete;
    ~SmartPtr() { delete p; }
    T& operator*() {
        return *p;
    }
};

void g(SmartPtr<int>& p) {
    if (*p == 0) {
        throw 1;
    }
}

void f(int x) {
    SmartPtr<int> w(new int(x));
    g(p);
}

int main() {
    f(5);
}
```

Это `unique_ptr`. Не может быть двух и более `unique_ptr`, указывающих на один и тот же `ptr`.

Идиома `RAII` распростроняется на файлы, а не только на указатели.

Есть еще `shared_ptr`, его можно копировать. Он хранит счетчик, сколько еще `ptr` указывает на то же, на что и он. 


## Exceptions in constructors

```cpp
#include <iostream>

struct A {
    A() { std::cout << "A"; }
    ~A() { std::cout << "~A"; }
};

struct S {
    A a;
    S(int x) {
        std::cout << "S";
        if (x == 0) throw 1;
    }
    ~S() {
        std::cout << "~S";
    }
};

int main() {
    try {
        S s(0);
    } catch (...) {

    }
}
```
Чьи деструкторы вызовутся, а чьи - нет, и в каком порядке?
`A` уничножилось, `S` - нет.


```cpp
#include <iostream>

struct A {
    A() { std::cout << "A"; }
    ~A() { std::cout << "~A"; }
};

struct S {
    A a;
    S(int x): a(new A()) {
        std::cout << "S";
        if (x == 0) throw 1;
    }
    ~S() {
        std::cout << "~S";
        delete a;
    }
};

int main() {
    try {
        S s(0);
    } catch (...) {

    }
}
```
Будет `memory lick`.
Если полем класса является указатель на что-то динамическое, то такое поле также следует обернуть в `smart_ptr`.


```cpp
#include <iostream>

struct A {
    A(int x) { 
        std::cout << "A"; 
        if (x == 0) throw 1;
    }
    ~A() { 
        std::cout << "~A"; 
    }
};

struct S {
    A* a;
    S(int x): a(new A(x)) {
        std::cout << "S";
    }
    ~S() {
        std::cout << "~S";
        delete a;
    }
};

int main() {
    try {
        S s(0);
    } catch (...) {

    }
}
```
Исключение вылетает в списке инициализации. 



```cpp
#include <iostream>

struct A {
    A(int x) { 
        std::cout << "A"; 
        if (x == 0) throw 1;
    }
    ~A() { 
        std::cout << "~A"; 
    }
};

struct S {
    A a;
    A aa;
    A aaa;
    S(int x): a(1), aa(0), aaa(2) {
        std::cout << "S";
    }
    ~S() {
        std::cout << "~S";
        delete a;
    }
};

int main() {
    S s(0);
}
```
Вызовется деструктор `A a`. (Все те, кто успели).
А как на уровне конструктора S обработать это? Хочется уметь писать `try` для списков инициализации. Такая возможность называется `function-try-block`:
```cpp
S(int x) try: a(1), aa(0), aaa(2) {
    std::cout << "S";
} catch (...) {
    std::cout << "caught";
}
```

Эта конструкция позволяет писать `try` у любой функции:
```cpp
void f() try {
    // ...
} catch (...) {
    // ... 
}
```

Вопрос к примеру выше: в каком состоянии находится переменная `S`? Правило следующее: `throw` автоматически дописывается в `catch`. К моменту входа уже уничтожены все родители и все созданные поля, и даже вызван ваш собственный деструктор, если вы вызывались из делегирующего конструктора.


## Exceptions in destructors

Начиная с `C++11` нельзя(с оговоркой) бросать исключения из деструктора, будет `terminate`.

```cpp
struct S {
    S(int x) {
        std::cout << "S";
    }

    ~S() {
        std::cout << "~S";
        throw 1;
    }
};

int main() {
    try {
        S s(0);
    } catch (...) {

    }
}
```

Если же прописать `noexcept(false)`, то `terminated` вызван не будет:
```cpp
struct S {
    S(int x) {
        std::cout << "S";
    }

    ~S() noexcept(false) {
        std::cout << "~S";
        throw 1;
    }
};

int main() {
    try {
        S s(0);
    } catch (...) {

    }
}
```


```cpp
struct S {
    S(int x) {
        std::cout << "S";
    }

    ~S() noexcept(false) {
        std::cout << "~S";
        throw 1;
    }
};

int main() {
    try {
        S s(0);
        throw 1;
    } catch (...) {

    }
}
```
В ситуации выше вызовется `terminate`: если исключение из деструктора брошено вместе с другим исключением.
Тем не менее, можем проверить, находимся ли мы в такой ситуации (летит ли еще исключение). То есть узнать, летит ли в данный момент `RunTime` летит исключение, или нет:
```cpp
struct S {
    S(int x) {
        std::cout << "S";
    }

    ~S() noexcept(false) {
        std::cout << "~S";
        if (!std::uncaught_exception()) {
            throw 1;
        }
    }
};

int main() {
    try {
        S s(0);
        throw 1;
    } catch (...) {

    }
}
```
`std::uncaught_exception()` - функция, реализованная компилятором. Она возвращает `bool`: true, если в данный момент летит исключение, false если иначе.

Начиная с `C++17`, эта функция устарела и ее заменили на `std::uncaught_exceptions()`. Сделано это было потому, что возможна ситуация, когда будет лететь несколько `exceptions` разом. 
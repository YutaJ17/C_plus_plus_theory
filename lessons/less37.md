`std::nothrow_t nothrow` для версии оператора `new`, которая не бросает исключения:
```cpp
struct nothrow {};

int* p = new(std::nothrow) int[100'000'000'000'000];  // non-throwing overloading
```

Бывает `new` с произвольным набором параметров:
```cpp
void* operator new(size_t n, int a, double b) {
    std::cout << n << "bytes allocated with custrom new " << a << ' ' << b << '\n';
    return malloc(n);
}

int main() {
    int* a = new int(1);
    int* p = new(1, 3.14) int(5);

    delete a;
    delete p;  // проблема, тривиальный delete неподходит кастомному new, будет UB
}
```

У оператора `delete` нет синтаксиса вызова с произвольным набором параметров, нужно прописать так:
```cpp
void* operator new(size_t n, int a, double b) {
    std::cout << n << "bytes allocated with custrom new " << a << ' ' << b << '\n';
    return malloc(n);
}

void operator delete(void* ptr, int a, double b) {
    std::cout << "custom delete call " << a << ' ' << b << '\n';
    free(ptr);
}

int main() {
    int* p = new(1, 3.14) int(5);
    operator delete(p, 1, 5.25);
}
```


Все ли корректно?
```cpp
struct S {
    S() {  throw 1; }
};

int main() {
    try {
        int* p = new(1, 3.14) int(5);
    } catch (...) {
        std::cout << "caught";
    }
}
```
`operator new` отработал, а конструктор кинул исключение. Как это работает?
Если вы в динамической памяти с помощью оператора `new` создаете какой-то объект, конструктор которого кидает исключение, то стандарт гарантирует вам, что за вас под капотом будет вызван оператор `delete` от таких же параметров, на том же адресе, а потом уже полетит исключение.

Что если сделать так:
Напоминание: статические неконстантные перменные внутри класса неопределяются, а начиная с `C++17` можно с помощью `inline`.
```cpp
struct S {
    inline static int count = 0;
    S() {
        ++count;
        if (count == 5) {
            throw 1;
        }
        std::cout << "created S\n";
    }
    ~S() {
        std::cout << "destroyed S\n";
    }
};

int main() {
    try {
        S* p = new S[10];
    } catch (...) {
        std::cout << "caught!\n";
    }
}
```
Все деструкторы штатно отработали, delete их вызвал. 

Вспомним наследование:
```cpp
struct Base {
    Base() { 
        std::cout << "created Base\n"; 
    }
    ~Base() { 
        std::cout << "destroyed Base\n"; 
    }
};


struct Direved : Base {
    int* p;
    Direved() { 
        p = new int;
        std::cout << "created Direved\n"; 
    }
    ~Direved() { 
        std::cout << "destroyed Direved\n"; 
        delete p;
    }
};


int main() {
    Base* b = new Derived;  // выделили память на Derived и проинициализировали указатель на Base
    delete b;
}
```
При `delete` должен вызваться деструктор, а после отчиститься память. Но поскольку деструктор Base невиртуальный, будет `memory lick`.


А если сделать так:
```cpp
struct Base {
    Base() { std::cout << "created Base\n"; }
    ~Base() { std::cout << "destroyed Base\n"; }
};


struct Direved : Base {
    Direved() { std::cout << "created Direved\n"; }
    ~Direved() { std::cout << "destroyed Direved\n"; }
};


int main() {
    Base* b = new Derived[10];
    delete[] b;
}
```
Все ли здесь нормально? Тоже некорректное поведение. Откуда `operator delete` знает, чей деструктор вызвать и сью память освободить? Будут вызываться деструкторы `Base`, а не `Derived`. Это вторая причина, зачем нужен виртуальный деструктор. Даже если у вас в классе нет выделения каких-то ресурсов, без виртуального деструктора не будет все правильно работать. 


```cpp
struct Base {
    Base() { std::cout << "created Base\n"; }
    virtual ~Base() { std::cout << "destroyed Base\n"; }
};


struct Direved : Base {
    int* p;
    Direved() { 
        p = new int;
        std::cout << "created Direved\n"; 
    }
    ~Direved() { 
        std::cout << "destroyed Direved\n"; 
        delete p;
    }
};


int main() {
    Base* b = new Derived[10];
    delete[] b;
}
```

Но с `delete[]` все равно не будет корректно работать, даже с виртуальным деструктором. Последний решает проблему с одиночными объектами, а для массивов стоит использовать массивы указателей или контейнеры.

---

Можно опеределить `opearator new()` для своего типа:
```cpp
struct S {
    inline static int count = 0;

    void* operator new(size_t n) {
        std::cout << "operator new for S\n";
        return malloc(n);
    }

    void opearator delete(void* ptr) {
        std::cout << "operator delete for S\n";
        free(ptr);
    }

    void* operator new[](size_t n) {
        std::cout << "operator new[] for S\n";
        return malloc(n);
    }

    void opearator delete[](void* ptr) {
        std::cout << "operator delete[] for S\n";
        free(ptr);
    }


    S() {
        ++count;
        if (count == 5) {
            throw 1;
        }
        std::cout << "created S\n";
    }
    ~S() {
        std::cout << "destroyed S\n";
    }
};

int main() {
    S* p = new S();
    S* t = new S[10];
    delete p;
    delete[] t;
}
```

Функции, помеченные макросом `_LIBCPP_WEAK`, могут переопределятся (нарушать ODR), как оператор new.

---

Представьте, что вы хотите запретить создания объекта S на куче, и разрешить только на стеке. Как это сделать? 
Прописать `=delete` у оператора `new`.
А как запретить создавать объекты на стеке, разрешив только на куче?
Один из способом: сделать объект с приватным деструктором, так как создание объекта на стеке подразумевает автоматическую возможность вызова из него. Проблема будет в том, что мы не сможем вызвать обычный `delete`.
В `C++20` добавили специальный оператор `delete` для решения это проблемы:
**class-specific usual destroying deallocation functions**
`void T::operator delete(T* ptr, std::destroying_delete_t);`

Также это можно сделать через обертку.


## Alignments and bit fields

```cpp
#include <iostream>

int main {
    int* a = new int[10];
    char* ac = reinterpret_cast<char*>(a);
    ++ac;
    int* b = reinterpret_cast<int*>(ac);

    *b = 1;
    int x = *b;
    std::cout << x;

    delete a[];
}
```
Указатель на `int` должен быть кратен 4. Код работает, но с санитайзерами - нет. Не все процессоры на уровне ассемблера прочитать такое.


`std::align` выравнивает pointer    
`alignof`
`alignas` спецификатор
`std::aligned_alloc` - malloc с конкретным выравниваем
Стандартный malloc, стандартный operator new возвращают указатель на память, выравненную на максимум для стандартных типов.
`std::max_align_t` - тип, у которого выравнивае - макимально возможное из стандартных типов


Битовые поля:
```cpp
struct S {
    unsigned char b1 : 3;
    unsigned char : 2;
    unsigned char b2 : 6;
};
```
Нужно для работы с бинарными данными. Например, чтение какого-то пакета с графикой или музыкой. Или обработка пакета по сети. 
В `allocator_traits` находятся метафункции, а также статические функции, чтобы аллокатору самому их не реализовывать.

Также в них есть `allocate` и `deallocate` для симметрии, чтобы все можно было делать через `traits`.
```cpp
using AllocTraits = std::allocator_traits<Alloc>;
```

И заменить все обращения к аллокатору на прямую (пример из реализации `vector`)
```cpp
alloc.deallocate(arr, cap);
```
к обращению через `allocator_traits`:
```cpp
AllocTraits::deallocate(alloc, arr, cap);
```

Так должно быть по стандарту у всех `stl containers`. 


`PoolAllocator`, `StackAllocator`, `FreeListAllocator` - примеры нестандартных аллокаторов, их нет в `stl`.


## Allocator-aware containers


Контейнер, "заботящийся" об аллокаторе. Такой контейнер должен думать еще о некоторых неочевидных вопросов, работая с аллокатором. Что еще должен уметь делать контейнер, чтобы считаться `aware`?
Что должен делать контейнер, когда он копируется, должен ли он копировать аллокатор или создавать новый по умолчанию? Иногда хочется разной реализации. А что такое копирование аллокатора? Это дешевая операция, то есть нужно сделать еще один аллокатор, указывающий на тот же пул. Но может быть мы хотим, чтобы при копировании контейнера создавался новый пул под этот контейнер. 
Контейнеру надо решить, делать копию аллокатора или сам аллокатор. Контейнер при своем копировании спрашивает у своего аллокатора, скопировать его или создать новый. 

Для этого существует специальный метод
```cpp
std::allocator_traits<allocator_type>::select_on_container_copy_construction
```
Может вернуть копию исходного аллокатора, или создать новый.

Аналогичный вопрос при присваивания контейнера, для этого есть метафункция
```cpp
std::allocator_traits<allocator_type>::propagate_on_container_copy_assigment::value
```
Является либо `std::true_type`, либо `std::false_type`.


Оператор присваивания для `vector`:
```cpp
vector& operator=(const vector& other) {
    T* newarr = AllocTraits::allocate(alloc, other, cap);  // выделили память под cap штук T
    size_t i = 0;
    try {
        for (size_t i = 0; i < other.sz; ++i) {
            AllocTraits::construct(alloc, newarr + i, other[i]);
        }
    } catch (...) {
        for (size_t j = 0; j < i; ++j) {
            AllocTraits::destroy(alloc, newarr + j);
        }
        AllocTraits::deallocate(alloc, newarr, other.cap);
        throw;
    }

    for (size_t i = 0; i < sz; ++i) {
        AllocTraits::destroy(alloc, arr + i);
    }
    AllocTraits::deallocate(alloc, arr, cap);
    arr = newarr;
    sz = other.sz;
    cap = other.cap;
    
}
```

Все гарантии исключений опираются на то, что ни `delete`, ни `destructor` исключений не бросают. 

Чтобы соблюсти `exception safety`, нужно сначала выделить новую память, а потом уже удалить старую, не наоборот.

В реализации выше по-прежнему есть проблемы. Мы не учитываем тот факт, что может нужно взять чужой аллокатор.
Надо либо присваивать аллокатор, или нет, в зависимости от `propagate`.

Мы должны старые объекты удалять старым аллокатором, а новые объекты выделить новым аллокатором, но при этом мы все еще должны старые объекты удалить позже, чем выделить новые. То есть нужно сохранить старый аллокатор.

```cpp
vector& operator=(const vector& other) {

    Alloc newalloc = AllocTraits::propagate_on_container_copy_assignment::value 
            ? other.alloc : alloc;

    T* newarr = AllocTraits::allocate(newalloc, other, cap);  
    size_t i = 0;
    try {
        for (size_t i = 0; i < other.sz; ++i) {
            AllocTraits::construct(newalloc, newarr + i, other[i]);
        }
    } catch (...) {
        for (size_t j = 0; j < i; ++j) {
            AllocTraits::destroy(newalloc, newarr + j);
        }
        AllocTraits::deallocate(newalloc, newarr, other.cap);
        throw;
    }

    for (size_t i = 0; i < sz; ++i) {
        AllocTraits::destroy(alloc, arr + i);
    }
    AllocTraits::deallocate(alloc, arr, cap);
    arr = newarr;
    sz = other.sz;
    cap = other.cap;
    alloc = newalloc;   // nothrow
    
}
```

А что если присваивание `alloc = newalloc` само кинет исключение? Это проблема решена радикальным образом в стандарте: оператор присваивания не кидает исключения (как и конструктор копирования).

Почему вы должны освобождать тем же аллокатором, каким выделяли? Потому что не все аллокаторы считаются равными. Когда вы пишите нестандартный аллокатор, ему нужно определить проверку на равенство. Она определяется по такому правилу: аллокаторы считаются равными, если одним из них можно удалить то, что было выделено другим.


Некоторые виды аллокаторы всегда равны друг другу. Допустим мы присвоили аллокаторы, и оказалось, что они равны (новый равен старому). Тогда нет смысла удалять новую память и выделять новую память и можно переиспользовать старую память. Значит, проверку на равенство аллокаторов нужно проверить в первую очередь.

А еще в `allocator_traits` есть метафункция `is_always_equal`. Все стандартные аллокаторы равны друг другу, и не нужно вызывать оператор сравнения аллокаторов, если можно в `CompileTime` проверить вызвать `is_always_equal` внутри `if constexpr`. 


## Перегрузка операторов new и delete

```cpp
/*
Container -> allocator_traits -> Allocator -> operator new -> malloc -> OS
*/
```

Все контейнеры взаимодействуют с new и delete только через аллокаторы. И если вы хотите высокоуровнево переопределить управление памятью для контейнеров, вы можете просто подменить аллокатор с помощью ООП.

Но можно подменить глобальный оператор new.

Оператор new состоит из двух частей, первая из которых занимается выделением памяти (функция operator new), а вторая - вызовом конструктора на выделенной памяти. И перегрузить можно только первую часть.
У `placement new` отсутствует первая часть. 

Оператор new - это не то же самое, что и функция оператор new. Вы можете переопределить функцию, а не оператор.
Мы переопределяем то, что происходит до вызова конструктора.

```cpp
void* operator new(size_t n) {
    std::cout << n << " bytes allocated\n";
    return malloc(n);
}

void operator delete(void* p) {
    free(ptr);
}

void operator new[](size_t n) {
    std::cout << n << " [] bytes allocated\n";
    return malloc(n);
}

void operator delete[](void* p) {
    free(ptr);
}

int main() {
    std::vector<int> v;
    for (int i = 0; i < 50; ++i) {
        v.push_bach(i);
    }

    int*p = new int[100];
    delete[] p;

}
```

`operator new[]` по умолчанию вызывает внутри себя `operator new()`.
А почему `vector` внутри себя вызывает `new()`, а не `new[]`? 

До этого мы реализовывали так (плохая практика):
```cpp
template <typename T>
struct allocator {
    T* allocate(size_t count) {  
        return reinterpret_cast<T*>(new char[count * sizeof(T)]);
    }
    // ...
};
```

Было через `operator new[]`, который вызывает `operator new[]` и конструктор определенного типа.

```cpp
template <typename T>
struct allocator {
    T* allocate(size_t count) {  
        return operator new(count * sizeod(T));
    }
    // ...
};
```

`return operator new()` - так можно попросить выделить память, не вызывая конструктор после.
`new (ptr) U(args.....)` - вызови конструктор, не выделяя памяти.

Функция `operator new[]` вызывается неявно, когда пишем `new T[]`, после чего вызываются конструкторы.


Как работает `operator delete`? Вызывается деструктор, а потом по этому указателю вызывается функция `operator delete`.
Как работает `operator delete[]`? Вызываются деструкторы в новом количестве, а потом по этому указателю вызывается функция `operator delete[]`.

А как оператор `delete[]` определяет, в каком количестве вызывать деструкторы? 



```cpp
void* operator new(size_t n) {
    std::cout << n << " bytes allocated\n";
    return malloc(n);
}

void operator delete(void* p) {
    free(ptr);
}

void operator new[](size_t n) {
    std::cout << n << " [] bytes allocated\n";
    return malloc(n);
}

void operator delete[](void* p) {
    free(ptr);
}

int main() {
    std::vector<int> v;
    for (int i = 0; i < 50; ++i) {
        v.push_bach(i);
    }

    std::cout << sizeof(std::string) << '\n';   // 32
    std::string* ps = new std::string[10];     // 328[] byte allocated, почему не 320?
    delete[] ps;

}
```

`operator new[]` записывает, для скольких нетривиальных объектов был вызван конструктор, чтобы `operator delete[]` знал, сколько вызвать деструкторов.


```cpp
template <typename T>
struct allocator {
    T* allocate(size_t count) { 
        return operator new(sizeof(T) * count);
    }

    void deallocate(T* ptr, size_t) {
        operator delete(ptr, size_t);
    }

    template <typename U, typename... Args>
    void construct(U* ptr, const Args&... args) {
        new (ptr) U(args...);  // проблема в лишнем копировании, см move semantics
    }

    void destroy(U* ptr) {
        ptr->U();
    }
};
```

Рассмотрим пример:
```cpp
struct Example {
    int* x;
    Example(): x(new int) {}
    ~Example() { delete x; }
};

int main() {
    Example* example = reinterpret_cast<Example*>(new char[sizeof(Example)]);   // OK
    new (example) Example();

    // Example* example = new Example[1];  // not OK, but why?

    example[0].~Example();
    operator delete(example);
}
```
- Выделили массив чаров размера Example и скастили к указателю на Example.
- По адресу Example вызвали конструктор.
- Вызвали деструктор.
- Очистили память.

А вот так:
```cpp
struct Example {
    int* x;
    Example(): x(new int) {}
    ~Example() { delete x; }
};

int main() {
    // Example* example = reinterpret_cast<Example*>(new char[sizeof(Example)]);   // OK
    //new (example) Example();

    Example* example = new Example[1];  // not OK, but why?

    example[0].~Example();
    operator delete(example);
}
```
Проблема в том, что мы передали в operator delete несдвинутый указатель example. `operator delete[]` неявно делает сдвиг на 8 байт, если тип был нетривиален (тот, у которого деструктор вызывать нужно).

Нужно исправить на `operator delete(example - 1);`  - все равно `UB`.
Чтобы такого избежать, нужно либо писать `reinterpret_cast`, либо использовать `new` и `delete` без `[]`.
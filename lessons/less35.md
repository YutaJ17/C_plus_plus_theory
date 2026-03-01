## Allocators and memory management

### Idea and basic usage of alloators

У всех шаблонных контейнеров stl последним параметром идет std::allocator по умолчанию в качестве аллокатора.

Есть `std::list<int>`, мы в него положили 1000000 int. На каждый `insert` вызывается `operator new`, что долго.

_Аллокатор_ - это класс, являющийся промежуточным звеном между контейнером и оператором `new`. Позволяет кастомизировать поведение контейнера при выделении памяти. 

```cpp
/*
container -> allocator -> operator new -> malloc -> OS
*/
```
Сейчас изучаем первую слева "->".

Напишем простейший аллокатор: `std::allocator`.
У аллокатора 4 основных метода:
- allocate (обращение к new)
- deallocate
- construct
- destroy

Так писать некорректно:
```cpp
template <typename T>
struct allocator {
    T* allocate(size_t count) {  // выделяем память под count объектов
        return reinterpret_cast<T*>(new char[count * sizeof(T)]);  // можно красивее, но об этом позже
    }

    void deallocate(T* ptr, size_t) {
        delete[] reinterpret_cast<char*>(ptr);
    }

    template <typename U, typename... Args>
    void construct(U* ptr, const Args&... args) {  // по адресу T конструируем объект
        new (ptr) U(args...);   // проблема в лишнем копировании
    }

    void destroy(T* ptr) {
        ptr->T();
    }
};
```


Исправим прошлую реализацию вектора:

```cpp
template <typename T, typename Alloc = std::allocator<T>>
class vector {
    T* arr;
    size_t sz;
    size_t cap;
    Alloc alloc;  // у каждого из контейнера есть свой аллокатор как поле

    template <bool IsConst>
    class base_iterator {
    public:
        using pointer_type = std::conditional_t<IsConst, const T*, T*> ptr;
        using reference_type = std::conditional_t<IsConst, const T&, T&> ptr;
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

        operator base_operator<true>() const {  
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

public:
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

    operator base_operator<true>() const {  
        return {ptr};
    }

    void reserve(size_t newcap) {
        if (newcap <= cap) {
            return;
        }

        T* newarr = alloc.allocate(newcap);

        size_t index = 0;
        try {
            for (; index < sz; ++index) {
                alloc.construct(newarr + index, arr[index]);
            }
        } catch (...) {
            for (size_t oldindex = 0; oldindex < index; ++index) {
                alloc.destroy(newarr + oldindex);
            }
            alloc.deadllocate(newarr, newcap);
            throw;
        }

        for (size_t index = 0; index < sz; ++index) {
            alloc.destroy(arr + index);
        }
        alloc.deallocate(arr, cap);

        arr = newarr;
        cap = newcap;
    }


    void push_back(const T& value) {}
};
```

### PoolAllocator
Сначала выделяет большой массив и им распоряжается. То есть весь memory management происходит локально, new вызвано 1 раз.

### StackAllocator
В конструктор аллокатора принимается массив.
Память выделяется на стеке:
```cpp
int main() {
    std::array<char, 1000000> arr;
    StackAllocator<int> alloc(arr);
    
    std::list<int, StackAllocator<int>> lst;  // list, хранящийся полностью на стеке
}
```

Возможна ситуация, когда аллокатору нужно аллоцировать не его тип. 
Стандартные контейнеры (например, `std::vector`) позволяют передать им свой аллокатор. Контейнер параметризован типом: `std::vector<int, MyAllocator<int>>`. Однако внутри `std::vector` может потребоваться выделить память не только под int, но и под какие-то свои внутренние служебные структуры (буферы, узлы списков, дескрипторы).

Вот тут и возникает ситуация: у нас есть аллокатор типа `MyAllocator<int>`, а нужно выделить память под объект типа Node (внутренний узел `std::list`).

```cpp
template <typename T, typename Alloc = std::allocator<T>>
class list {
    struct BaseNode {
        BaseNode* prev;
        BaseNode* next;
    };

    struct Node : BaseNode {
        T value;
    };

    BaseNode fakeNode;
    size_t count;
    Alloc alloc;
};
```

В аллокаторе есть такая метафункция, которая позволяет получить такой же аллокатор, но от другого T. Она называется `rebind`:
```cpp
template <typename U>
allocator(allocator<U>) {}

template <typename U>
struct rebind {
    using other = allocator<U>;
};
```

Тогда, возвращаясь к `std::list`:
```cpp
template <typename T, typename Alloc = std::allocator<T>>
class list {
    struct BaseNode {
        BaseNode* prev;
        BaseNode* next;
    };

    struct Node : BaseNode {
        T value;
    };

    BaseNode fakeNode;
    size_t count;
    typename Alloc::rebind<Node>::other alloc;  

    list(const Alloc& alloc): fakeNode(), count(), alloc(alloc) {}
};
```
Главная проблема: мы должны хранить и уметь конструировать аллокатор другого типа.

Почему бы аллокатор не сделать шаблонным шаблонным аргументом?
```cpp
template <typename T, template<typename> Alloc = std::allocator>
class list {
    struct BaseNode {
        BaseNode* prev;
        BaseNode* next;
    };

    struct Node : BaseNode {
        T value;
    };

    BaseNode fakeNode;
    size_t count;
    typename Alloc::template rebind<Node>::other alloc;  

    list(const Alloc& alloc): fakeNode(), count(), alloc(alloc) {}
}
```
К сожалению, нельзя, потому что у аллокатора могут быть больше разных шаблонных параметров, и не факт что это типы, а не числа.


Аллокатор не влияет на размер объекта благодаря `empty base optimization`:
```cpp
class vector : private Alloc {};
```

А начиная с `C++20` есть более умный способ этого добиться:
```cpp
template <typename T, typename Alloc = std::allocator<T>>
class vector {
    T* arr;
    size_t sz;
    size_t cap;
    [[no_unique_address]] Alloc alloc;

    template <bool IsConst>
    class base_iterator { 
        // ... 
    }
};
```


## Stateful_allocators, allocator_traits

### FreeListAllocator
Когда вы просите Node, он их выделяет, а когда просите освободить, он их не освобождает, а складывает в собственный List свободных Node. И когда вы просите снова, если у него есть готовые свободные, он выдает свои, а не идет в new.

--- 

Как скопировать аллокатор? Мы хотим, чтобы он ссылался на тот же массив, но не копировал этот массив (в случае Pool Allocator).
Allocator - это очень легковесная вещь, как и iterator.


### allocator_traits

Структрура над аллокатором, которая для всех аллокаторов предопределяет некоторые using, некоторые функции, чтобы в каждом аллокаторе не приходилось писать их заново.
То есть в стандартном аллокаторе нет тех четырех базовых методов, они все лежат в `allocator_traits`. И стандартные контейнеры обязаны обращаться не к аллокатору напрямую, а к `allocator_traits`.

```cpp
template <typename Alloc>
struct allocator_traits {
    template <typename U, typename... Args>
    static void construct(Alloc& alloc, U* ptr, const Args&... args) { 
        if constexpr(/*alloc has method construct*/) {
            alloc.construct(ptr, args...);
        } else {
            new (ptr) U(args...); 
        }
    }
};
```

- `rebind_alloc<T>`	Alloc::rebind<T>::other if present, otherwise SomeAllocator<T, Args> if this Alloc is of the form SomeAllocator<U, Args>, where Args is zero or more type arguments

- `rebind_traits<T>`	`std::allocator_traits<rebind_alloc<T>>`


Дополним схему:
```cpp
/*
Container -> allocator_traits -> Allocator -> operator new -> malloc -> OS
*/
```

Зная это, также следует исправить реализацию reserve в векторе:
вместо
```cpp
T* newarr = alloc.allocate(newcap);
```
написать
```cpp
T* newarr = std::allocator_traits<alloc>::allocate(alloc, newcap);
```


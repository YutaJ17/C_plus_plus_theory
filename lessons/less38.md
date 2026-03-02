## Scoped allocators

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <scoped_allocator>

template <typename T>
using MyAlloc = std::allocator<T>;

int main() {

    using MyString = std::basic_string<char, std::char_traits<char>, MyAlloc<char>>;

    MyAlloc<MyString> alloc;

    std::vector<MyString, MyAlloc<MyString>> v(alloc);

    v.push_back("abc");
    v.push_back("cde");
}
```
У строк тоже может быть аллокатор. На самом деле, string - это 
alias для `std::basic_string<char, std::char_traits<char>, std::allocator<char>>`.

Есть еще одна проблема, связанная с аллокаторами. Допустим, мы захотели создать строку на кастомном аллокаторе. Предположим, что `MyAlloc` - какой-то кастомный аллокатор, например, PoolAllocator.  
А дальше, захотелось создать вектор из этих стрингов, притом вектор тоже будет на нестандартном аллокаторе. 

Это называется проблемой `scoped` аллокаторов: если я завожу контейнер из других контейнеров и хочу, чтобы все выделялось на нестандартном аллокаторе, то нужно, чтобы у вектора был нестандартный аллокатор, и у каждой из строк был нестандартный аллокатор.
Вроде в коде выше все так и реализовано. В чем тогда заключается проблема? И вектор, и строки в нем создадутся на кастомном аллокаторе, но это будут разные экземляры аллокатора. Ведь когда вектор будет добавлять в себя строку, он же не будет знать, что ему нужно взять тот же объект аллокатора, от которого создан он сам. 

А хочется пользоваться контейнером, который использует один и тот же объект (экземпляр) аллокатора и для себя самого, и для своих подобъектов. Для решения этой проблемы в стандартной библиотеке есть класс 
`scoped_allocator_adaptor`.

Надо написать так:
```cpp
#include <iostream>
#include <vector>
#include <string>
#include <scoped_allocator>

template <typename T>
using MyAlloc = std::allocator<T>;

int main() {

    using MyString = std::basic_string<char, std::char_traits<char>, MyAlloc<char>>;

    MyAlloc<MyString> alloc;

    std::vector<MyString, std::scoped_allocator_adaptor<MyAlloc<MyString>>> v(alloc);

    v.push_back("abc");
    v.push_back("cde");
}
```

Теперь этот вектор будет создавать все строки на одном и том же экземляре аллокатора, даже если мы явно не указали, на каком объекте аллокатора создавать очередную строку. 
Мы можем хотеть, чтобы в `std::vector<std::list<std::map<T>>>` у `vector` был свой аллокатор, у `list` - свой, у `map` - свой. Тогда мы можем в `scoped_allocator_adaptor` передать эти типы, по одному на каждый уровень. 

Как это реализовано?
```cpp
template <typename T>
struct scoped_allocator_adaptor {
    Alloc alloc;
};
```
Метод `construct` должен создать объект T, дополнительно передав экземпляр аллокатора в то, что он создает.

```cpp
template <typename T>
struct scoped_allocator_adaptor {
    Alloc alloc;

    template <typename T, typename... Args>
    void construct(T* ptr, const Args&... args) { // пока принимаем по const lvalue ссылке, по-другому не умеем
        using InnerAlloc = typename T::allocator_type;
        alloc.construct(ptr, args..., InnerAlloc(alloc));
    }
};
```

С помощью `uses allocator` мы проверяем, умеет ли тип принимать аллокатор, и если не умеет, то мы не даем ему аллокатор. 

```cpp
template <typename T>
struct scoped_allocator_adaptor {
    Alloc alloc;

    template <typename T, typename... Args>
    void construct(T* ptr, const Args&... args) {  // очень упрощенная реализация
        if constexpr (std::uses_allocator_v<T, Alloc>) {
            using InnerAlloc = typename T::allocator_type;
            alloc.construct(ptr, args..., InnerAlloc(alloc));
        } else {
            alloc.construct(ptr, args...);
        }
    }
};
```

Есть еще `polymorphic allocators`, но об этом позже.


## Атрибуты

`attribute specifier sequence`
Можно помечать переменные, функции, а иногда и аргументы функции атрибутами.
Синтаксис:
```cpp
[[attribute-list]]
[[using attribute namespace : attribute-list]]
```

### Some of standard attributes

- `[[async]]`
- `[[no_unique_address]]` - разрешить объекту не заводить отдельный адрес (замена empty base optimization)
- `[[nodiscard]]` - ставится перед функциями (обычно), и говорит компилятору, что нужно выдать `warning`, если результат этой функции дискардится (не используется).
Для каких функций следует писать такой атрибут? Например: operator new, operator new[], allocate, empty.
По хорошему, 2 случая использования:
1. Если результат функции опасно игнорировать (будет утечка)
2. Если название этой функции таково, что оно может ввести в заблуждение, что она что-то меняет (empty)

- `[[deprecated]]`
- `[[myabe_unused]]` - перед аргументами функций, говорит, что аргумент может не использоваться в теле функции.
Нужен для шаблонного кода: в зависимости от T аргумент может быть как использован, так и нет.
- `[[likely]]` - ставится после if, подсказывая компилятору более вероятную ветку if
- `[[unlikely]]`
- `[[assume]]` - помогает в оптимизации: `[[assume(x<100)]]`


## Move semantics and rvalue-references

### Idea of move-semantics, Rule of Five

До `C++11` в языке была очевидная неоптимальность, которую хочется убрать.

```cpp
class vector {
public:
    void push_back(const T& value) {}
    // ...
};

int main() {
    vector<string> v;
    v.push_back("abc");
}
```
Когда мы так пишем, мы инициализируем `const T& value` сишной строкой "abc" (`const char*`). Что в этот момент происходит? Создается временная строка, сишная строка конвертируется в объект `std::string`, к которому привязывается константная ссылка, как к временному объекту. 
Дальше в `push_back` в какой-то момент написано: `new(arr + n) T(value);` T - `string`, value - это тоже `string`. То есть создается второй `string`.
У нас создался первый `string` для того, чтобы быть переданным в `push_back`, а потом из него создался второй `string`, который положился в вектор, а временный `string` уничтожился. 
Главная проблема в том, что нет способа этого избежать. Как научится класть в вектор объекты так, чтобы их второй раз не копировать? 
И даже если написать `push_back(string("abc"));`, это ничему не поможет, будет то же самое. И все равно будет 2 создания `string`. 

Есть такая функция `emplace_back`. Она принимает не объект T, а аргументы, из которых она создает объект на месте. Эта функция была придумана для решения этой проблемы.
```cpp
template <typename... Args>
void emplace_back(const Args&... args) {
    new(ptr) T(args...);
}
```
Реальный объект `string` создался только один. Но на самом деле глобально проблему это не решает. Мы просто спустили проблему на уровень глубже. А что если у нас `vector<vector<vector<string>>>`? Будет создавать кучу промежуточных объектов и потом их копировать. 
Очевидно, это проблема не только добавления в контейнер, но и любого сохранения данных. 
```cpp
struct S {
    string data;
    S(const string& str): data(str) {}
};

int main() {
    S("abc");  // создатся промежуточная строка для привязки abc к константной ссылке
}
```

И еще одна проблема: с исключениями.
```cpp
string s("abc");
throw s;  // также происходит копирование, в динамическую память
```

Мы хотим, чтобы в таких ситуациях можно было не копировать объекты, а делать что-то более умное.

Можно для своих классов определить не `copy constructor`, а `move constructor`. 
Хотим уметь создавать объекты из других объектов не полностью копируя их, а забирая у них данные и оставляя их пустыми. Вместо `O(n)` будет `O(1)`.

Для этого нужно ввести новую операцию над объектами. И научить компилятор, когда он должен эти move операции использовать, а когда нет.

```cpp
class string {
    char* arr;
    size_t sz;
    size_t cap;

public:    
    string(string&& other) {

    }
};
```
Пока просто поверим, что если написать `&&`, то все будет как надо. 

```cpp
class string {
    char* arr;
    size_t sz;
    size_t cap;

public:    
    string(string&& other) : arr(other.arr), sz(other.sz), cap(other.cap) {
        other.arr = nullptr;
        other.sz = other.cap = 0;
    }

    string& operator=(string&& other) {
        delete[] arr;
        arr = other.arr;
        other.arr = nullptr;  // можно короче через move-and-swap idiom
        cap = other.cap;
        other.cap = 0;
        sz = other.sz;
        other.sz = 0;

        return *this;
    }
};
```
Тот объект, из которого мувнули, не перестает быть валидным, ему можно что-то присвоить.

Остался вопрос: в какой момент должны вызываться этот конструктор?

**Rule of Five**: если в классе есть нетривиальный (один из пунктов):
- copy constructor
- copy assignment operator
- destructor
- move constructor
- move assignment operator

, значит все 5 должны быть нетривиальными.

Для примитивных типов move операции ничем не отличаются от copy. Если в классе не упомянуты move специальные методы, то будут вызыватся их копирующие аналоги. 
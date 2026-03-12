Часто встречались ситуации, когда в одну сторону неявный каст разрешен, а в другую - нет.

```text
type& <--- type
          /
         /
        /
       V
const type& <--- const type

type& <--- (запрещен, нужен const_cast) const type
```

```text
Derived& <--- Derived
          /
         /
        /
       V
    Base& <--- Base

Derived& <--- (запрещен, нужен static_cast) Base 
```

Для категорий также:

```text
    type&& <--- rvalue
                /
               /
              /
             V
        const type& <--- lvalue

type&& <--- (запрещен, нужен std::move) lvalue
```

```cpp
int x = 5;
int&& y = std::move(x);
int&& z = y; // CE
int&& z = std::move(y);
int t = 6;
int z = t;
```

`rvalue ref`, как и `const lvalue ref` умеют продлевать жизнь объектам
```cpp
int&& r = 1; // можно менять
const int& r1 = 2;  // нельзя менять
```

А как оно сочетается с константностью?
Правила констанстности действуют независимо от правил вида `value`.

```cpp
int x = 5;
const int&& y = std::move(x);
int&& z = std::move(x); // CE, нужен const cast
```

```cpp
const int&& r = 1;  // ok
```

```cpp
template <typename T>
T&& move(const T& x) {  // CE, нарушение константности
    return static_cast<T&&>(x);
} 
```

```cpp
void f(const T& x) {
    g(std::move(x));  // копирование
}
```

### reference qualifires

```cpp
struct S {
    string str;
    string getData() const {
        return str;
    }
};

int main() {
    S{"abc"}.getData();  // копирование
}
```
Иногда хочется по-другому работать с объектом, если `this` - это `rvalue`.
Раз существует перегрузка между `const` и `non-const` объектами, было бы неплохо иметь перегрузку между `rvalue` и `lvalue` объектами (на примере, чтобы `getData()` делала мув).

Для этого есть `reference` квалификаторы:
```cpp
string getData()&& {  // non-const rvalue
    return std::move(str);
}

string getData() const {  // для любых объектов, const, non-const, lvalue, rvalue
    return str;
}


string getData() & {  // non-const lvalue
    return str;
}


string getData() const & {  // const lvalue
    return str;
}
```

Возможность присваивания у `operator=` нужно задавать через `&`.


### Forwarding references, std::forward

```cpp
void push_back(const T&) {
    new (ptr) T(value);
}

void push_back(T&&) {
    new (ptr) T(std::move(value));
}
```
`construct` в аллокаторе:
```cpp
void construct(U* ptr, const Args&... args) {}  // переменное число аргументов, 2 перегрузки не спасут, нужно 2^n перегрузок, где 
// n - количество принимаемых аргументов
```

`emplace_back`:
```cpp
template <typename ...Args>
void emplace_back(const Args&... args) {}  // переменное число аргументов, 2 перегрузки не спасут, нужно 2^n перегрузок
```

Сейчас проблема решена, если стек вызовов состоит из 1 вызова функций, то проблема решена. Но если нам нужно пойти дальше и передать аргументы дальше, в следующую функцию, с сохранением вида `value`, то проблема не решена.

Здесь прийдется ввести новый вид ссылок и новую функцию. Как принять переменное число аргументов, сохранив их вид `value`?

```cpp
template <typename... Args>
void emplace_back(Args&&... args) {}
```

Работоспособность данного феномена обеспечивает `std::forward`. 

```cpp
void construct(U* ptr, Args&&... args) {
    new (ptr) U(std::forward<Args>(args)...);
}
```

Утвержается, что если мы принимаем аргументы через `Args&&`, функция `std::forward` передает аргументы дальше, сохраняя `value`.

Почему в `Args&&` можно передавать `lvalue`, ведь согласно сказанному в прошлых лекциях, это не должно работать? 
Все касаемо `rvalue` ссылок было верно, за одним исключением: если такая ссылка является шаблонным параметром функции, то правило неверно. 

Если такая ссылка `T&&` является типом аргумента функции, где `T` - шаблонный параметр данной функции, то такую ссылку можно инициализировать не только через `rvalue`. (Это костыль стандартной библиотеки, сделанный ради решения проблемы выше).

Такие ссылки называются _универсальными_(universal/forwarding references).


```cpp
struct S {
    vector<string> v;

    template <typename ...Args>
    S(Args&&... args) {
        // static_assert(), что все типы конвертируемы в string
        v.reserve(sizeof...(args));
        (v.push_back(std::forward<Args>(args)), ...);  // fold expression
    }
};
```

Универсальные ссылки имеют еще одно свойство: если по ней передать `lvalue`, то меняются правила вывода типов шаблона.
```cpp
template <typename T>
void h(T&& x) {}

template <typename T>
void g(T& x) {}

template <typename T>
void f(T x) {}

int main() {
    int x = 0;
    f(x);  // T = int
    g(x);  // T = int
    h(x);  // T = int&    (важное правило)

    int& y = x;
    f(y);  // T = int
    g(y);  // T = int
    h(y);  // T = int&

    int&& z = std::move(x);
    f(z);  // T = int
    g(z);  // T = int 
    h(z);  // T = int&


// вызов от rvalue
    h(1);  // T = int
    h(std::move(x)); // T = int
}
```
Правило: если функция принимает универсальную ссылку, то при вызове от `lvalue` навешивается `&`. 


**Reference collapsing rules**: если в момент вывода типа переменной в шаблоне накладываются амперсанды один на другой, то одни преобразовываются по следующему правилу:
`&` + `&` = `&`
`&&` + `&` = `&`
`&` + `&&` = `&`
`&&` + `&&` = `&&`

Благодаря этому правилу (костылю), универсальные ссылки могут принимать как `lvalue`, так и `rvalue`, причем если они принимают `lvalue`, компилятор искусственно добавляет `&` на `T`. 
Обобщим: если принимаем
- `lvalue`, то навешивается `&` на `T`
- `rvalue`, то навешивается `&&` на `T`

Это наводит на мысль, как работает функция `std::forward`.


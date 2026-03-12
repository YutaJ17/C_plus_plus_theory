Бывают типы, которые можно мувать, а копировать нельзя. Например, `std::unique_ptr`.

Остался вопрос, как компилятор понимает, когда вызывать move операцию, а когда копирующую?
Короткий ответ: если компилятор видит вызов от `rvalue`, он будет пытаться вызвать мувающую операцию, а от `lvalue`- копирующую.

### Mystic function std::move

Можно принудительно заставить компилятор вызвать мув операцию, несмотря на то, что переменная была `lvalue`.

```cpp
struct S {
    string data;
    S(const string& data) : data(std::move(data)) {} // все равно будет копирование из-за const
};
```
Стоит сделать так:
```cpp
struct S {
    string data;
    S(const string& data) : data(data) {}   // для lvalue - копирование
    S(string&& data) : data(std::move(data)) {}  // для rvalue - перемещение
};
```

Роль `std::move` в том, чтобы перенаправить перегрузку в мув версию.

Теперь для вектора:
```cpp
void push_back(const T& value) {
    new(ptr) T(value);
}
void push_back(T&& value) {
    new(ptr) T(std::move(value));
}
```

Проблема с бросанием исключений (лишнее копирование) решена в стандарте более приоритетным вызовом мув.
```cpp
string s = "abc";
throw s;
```

Функция `swap` (плохая реализация):
```cpp
template <typename T>
void swap(T& x, T& y) {
    T tmp = x;
    x = y;
    y = tmp;
}
```

С помощью мув семантики:
```cpp
template <typename T>
void swap(T& x, T& y) {
    T tmp = std::move(x);
    x = std::move(y);
    y = std::move(tmp);
}
```

Реализация функции std::move, правильно работающая в 90% случаев (наивная реализация):
```cpp
template <typename T>
T&& move(T& x) {
    return static_cast<T&&>(x);
}
```
`std::move` - это явный каст к `rvalue`.

Что произойдет с s?
```cpp
string s = "abc";
std::move(s);
```
Ответ: ничего.

Эта функция имеет смысл только на этапе компиляции. Нужна только для того, чтобы компилятор перенаправился в другую версию перегрузки.


```cpp
template <typename T>
T&& move(T x) {   // копирование
    return static_cast<T&&>(x);   // битая ссылка
}
```

```cpp
template <typename T>
T move(T& x) {   // лишний вызов мув конструктора
    return static_cast<T&&>(x);
}
```

### Formal definitions of lvalue and rvalue

`lvalue` and `rvalue` are categories of expressions, not types !!!
Each *expression* has two independent properties: *type* and *value category*.

**lvalue**
1. id (идентификатор - частный случай expression)
Любая переменная  - `lvalue`, неважно, какой у нее тип.

2. `"abc"` (строковые литералы)

3. `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `<<=`, `>>=`, `|=`, `&=`

4. `++expr`, `--expr`

5. `*ptr`, `a[i]`

6. comma if rhs is lvalue

7. `?:` if both operands are lvalue

8. function call (or cast expression) - lvalue, if return type is T&



**rvalue**
1. литералы (кроме строковых)
`5`, `a`, `2.of`, `true`

2. `+`, `-`, `*`, `/`, `%`, `>>`, `<<`, `&`, `|`, `==`, `!=`, `&&`, `||`

3. `expr++`, `expr--`

4. `&a`

5. comma if rhs if rvalue

6. `?:` if both operands are rvalue

7. function call (or cast expression) - rvalue, if return type is T or T&&

 
### Rvalue-references and their properties

`T&&`
2 главных свойства (отличающих `rvalue ref` от обычной ссылки):
1. будучи возвращенной с функции она является `rvalue expression`
2. проинициализировать ее можно только `rvalue` выражением

На примере `int`:
```cpp
int x = 5;
int&& y = x;  // CE
int&& y = 6;  // ok, life time prolongation (или extension) (продление жизни объекта)
y = 7;  // ok 
int&& z = y; // CE, так как справа - lvalue 
int&& z = std::move(y);  // ok, так как из std::move возвращается T&&, то есть rvalue expression
int&& t = static_cast<int&&>(x);  // ok
t = 1; // ok

```
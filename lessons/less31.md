## ranged-base for

```cpp
int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};
    for (int& x: v) {  // ranged-base for
        v.push_back(x);  // очевидно, UB
    }
    for (int& x: v) {
        std::cout << x;
    }
}   
```
Почему `UB`?
`ranged-base for` - синтаксический сахар, который разворачивается в проход от итератора до итератора. По сути, `int& x` - разыменованный итератор. Поскольку итераторы в векторе инвалидируются при `push_back()` в него, следующий шаг `for` - уже `UB`.


```cpp
int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};
    for (std::vector<int>::iterator it = v.begin(); it != v.end(); ++it) {   // segfault
        v.push_back(x);  
    }
    for (int& x: v) {
        std::cout << x;
    }
}   
```
`end` каждый раз меняется.

Почему 2 реализации выше работают по-разному? Потому, что они не эквивалентены.
На самом деле, `ranged-base for` запоминает изначальные `begin` и `end`. Зачем он их запоминает? Потому что каждый раз вызывать `end` - довольно дорогостоящее занятие.

Формально, делать тоже самое с `deque` - тоже `UB`.


## streams 

```cpp
#include <iostream>

int main() {
    std::cerr << "abc";  // поток ошибок, удобно для логгирования ошибок
    std::cout << "hi";
}
```
Вывело: "abchi".


```cpp
#include <iostream>

int main() {
    std::cout << "abc";
}
```
У программы будет полезный вывод ("hi") и вывод ошибок ("abc").
Как сделать так, чтобы эти потоки оказались разделены?
Например, один из потоков направить в файл, а другой - в консоль?

Для этого при запуске исполняемого файла следует прописать `./a.exe > output.txt`.
Но если так делать несколько раз, сообщение перезатирается (перезаписывается).
Чтобы дописывать в имеющийся файл: `./a.exe >> output.txt`.

А как работать с несколькими потоками?

У потоков есть номера (факт из Linux):
0. cin
1. cout
2. cerr

Можем прописать при запуске программы: `./a.exe 1>output.txt`.
`./a.exe 2>output.txt`
`./a.exe 1>output.txt 2>err.txt`

А если мы хотим слить потоки в один файл?
`./a.exe 1>output.txt 2>&1`   - говорим, куда 1, а потом, что 2 - туда же.

file1.cpp
```cpp
#include <iostream>

int main() {
    int x;
    std::cin >> x;
    std::cout << x + 5; 
}
```

file2.cpp
```cpp
#include <iostream>

int main() {
    std::cout << 123;
}
```

>> g++ file1.cpp -o file1.exe
>> ./b.out
>> 6    
>> 11
>> g++file2.cpp -o file2.exe
>> ./a.out
>> 123
>> ./file2.out | ./file1.out
>> 128

`|` оператор **pipe**
`a.out` вывел 123 и направил его на вход в `b.out`.

>> grep "vector" 8_1_vector.cpp
Программа выведет все строки файла с этим словом.

>> ls -lR . | grep cpp
Можно искать регулярные выражения.

Команды `head -5 8_1_vector.cpp` (`head -n 5 8_1_vector.cpp` или же `head 8_1_vector.cpp`)
и с `tail` по аналогии выводят первые/последние `n` строк файла.

>> ls -l . | head

У ls можно сортировать вывод по размеру/дате изменения:
>> ls -lS . | head
>> ls -lt . | head

wc - word count, считает количество слов(по умолчанию) или строк
>> ls -l | wc
>> ls -l | wc -l

tee - позволяет раздвоить вывод
>> ./a.out | tee out.txt

Позволяет, например, и в файл/файлы записывать, и в консоле выдеть, что записываешь.

/dev/bull - виртуальный файл линукса, данные из которого удаляются. Можно подавлять вывод.
>> ./a.out | tee out.txt >/dev/null

---

### Буфер

```cpp
#include <iostream>
#include <cassert>

int main() {
    std::cout << "123";
    assert(false);    // тригерим RunTimeError
}
```
`123` не вывелось. Когда мы что-то выводим в `cout`, оно не сразу выводится. `cout` - это объект, который хранит в себе некоторый буфер (массив чаров), точнее, указатель на `stream buffer`. И реальный вывод на консоль происходит при заполнении буфера. 
Стандартная библиотека написана так, чтобы к операционной системе ходить пореже.
Чтобы до падения что-то успело вывестись, нужно отчищать буфер:

```cpp
#include <iostream>
#include <cassert>

int main() {
    std::cout << "123";
    std::cout.flush(); // долгая операция
    assert(false);    // тригерим RunTimeError
}
```

```cpp
#include <iostream>
#include <cassert>

int main() {
    std::cout << "123" << std::endl;
    assert(false);    // тригерим RunTimeError
}
```
`std::endl` - специальный объект, который, будучи выведенным в `cout`, выводит `\n` и делает `flash()`.


### метод tie

```cpp
#include <iostream>
#include <cassert>

int main() {
    std::cout << 123;
    int x;
    std::cin >> x;
    std::cout << x + 5;
}
```
Между `cout` и `cin` происходит `flash()`. Почему так? Потому, что по умолчанию эти объекты связаны. Объект `cout` хранит в себе не только буффер, но и указатель на `sin`. А `sin` хранит указатель на `cout`. Зачем так сделано? Это нужно для корректного порядка ввода-вывода. Эти потоки можно отвязать. 

```cpp
#include <iostream>

int main() {
    std::cin.tie(nullptr);
    std::cout.tie(nullptr);

    std::cout << 123;
    int x;
    std::cin >> x;
    std::cout << x + 5;
}
```

Еще, по умолчанию, `cout` и `cin` синхронизированы со стандартными C-потоками ввода/вывода, работающие через `scanf` и `printf` (которые тоже буферизованы). Это тоже можно отключить.

```cpp
#include <iostream>

int main() {
    std::cout.tie(nullptr);
    std::cin.tie(nullptr);
    std::cin.sync_with_stdio(false);
    std::cout.sync_with_stdio(false);

    int x;
    std::cout << 123;
    std::sin >> x;
    std::cout << x + 6;
}
```
Жертвуем гарантией безопасности ради скорости. 

В какой момент создаются и уничтожаются `cin` и `cout`? Это глобальные переменные, тип которых - `std::istream` и `std::ostream` соответственно. Как и у любых глобальных, их конструкторы вызываются 
до `int main()`, а вызываются - после.

А если глобальный объект уже использует `cout`?

```cpp
struct A {
    A() {std::cout << "A"}
};

A a;
```
Все ок, ведь `#include <iostream>` написан выше.

### fstream, sstream

```cpp
#include <fstream>

int main() {
    std::ifstream in("input.txt");  // создали файловый поток, привязанный к файлу
}
```


```cpp
#include <sstream>
int main() {
    std::string str("1 2 3 4 5");
    std::istringstream iss(str);

    int x;
    iss >> x;
    std::cout << x + 5;
}
```
Логично, что работает быстрее, чем `cin`, так как нет взаимодействия с операционной системой.
Нужен для имитации ввода/вывода в поток, например, писать тесты к алго задачам.
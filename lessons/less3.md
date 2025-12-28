
Любая программа на C++ - последовательность объявлений.

Мы можем объявлять:
1. переменные
2. функции
3. классы
4. структуры
5. пространства имен
6. юнионы

Внутри функции мы пишем _statements_. Они могут быть как объявлениями, так и чем-то другим. 


## Конфликты имен переменных

```cpp
#include <iostream>

int a;  // глобальная переменная, иницилизируется 0
void f(int x);  // void - ничего не возвращает

class C;
struct S;
enum E;
union U;
namespace N {
	int x;
}
namespace N {
	int y;
}

int main() {
	using N::x;
	std::cout << x; // ошибки не будет
}
```

`namespace` - пространство имен, позволяет более удобно организовывать код, чтобы случайно не было совпадений имен, относящихся к разным логическим кускам кода

```cpp
namespace N {
	int x;
}

using N::x;

int main() {
	std::cout << x;
}
```


```cpp
namespace N {
	int x;
}

namespace P {
	int y;
}

using namespace N;  // делает все сущности из namespace N на глобальном уровне

int main() {
	using namespace P;  // делает все сущности из namespace Y в функции main
	std::cout << x;
	std::cout << y;
}
```

Не рекомендуется писать `using namespace std`, возможен конфликт имен функций, например при вызове своей функции `distance` будем использовать функцию из `std`, которая тоже там есть.


```cpp
int main() {
	int x;
	using std::cout;  // вот так можно
	cout << x;
}
```

```cpp
int main() {
	using vi = std::vector<int>; // создаем новое название для std::vector в области видимости функции main
}
```

```cpp
template <typename T>
using v = std::vector<T>;
```

```cpp
typedef_long_long ll; // лучше не использовать, это legacy от C
```


- global scope
- namespace scope
- class scope 
фигурные скобки задают scope

```cpp
x = 0;
int main() {
	int x = 1;
	for (int i = 0; i < 10; i++) {
		int x = i;
		std::cout << x;  // не ошибка, локальный x затмевает глобальный
		std::cout << ::x;  // глобальный x
	}
}
```

```cpp
int main() {
	for (int x = 0; x < 10; x++) {
		int x;  // error: redeclaration
		std::cout << x;
	}
}

```

В одном и том же scope объявить переменную с тем же именем нельзя.

так тоже можно:
```cpp
int main() {
	int x = 5;
	{
		int x = 3;
		std::cout << 3;
	}
}
```

обращение к переменной неодназначно
```cpp

namespace N {
	int x = 7;
}

namespace NN {
	int x = 6;
}


int main() {
	using namespace N;
	using namespace NN;
	std::cout << x;  // reference is ambigous
}
```

```cpp

namespace N {
	int x = 7;
}

namespace NN {
	int x = 6;
}


int main() {
	using namespace N;
	using NN::x;  // более приоритетный x
	std::cout << x;  // ошибки нет
}
```

`error: redeclared as different kind of entity`
```cpp
namespace NN {
	int x = 6;
}

int main() {
	using NN::x;
	using x = std::vector<int>;
	
	std::cout << x;
}
```

а вот так корректно:
```cpp
namespace NN {
	int x = 6;
}

int main() {
	using NN::x;
	{
		using x = std::vector<int>;
	}
	std::cout << x;
}
```


`point of declaration` - место в коде, начиная с которого имя действует
```cpp
int x = 0;
int main() {
	int x = x;  // равносильно int x;
	x = 5;
	std::cout << x; // 5
}

```

## Правило одного определения (ODR)

_One Definition Rule_ - каждая используемая сущность в программе должна быть один раз определена.

Класс можно определить несколько раз при условии, что все определения дословно идентичны. 

```cpp
class C {};  // {} - определение
```

С функциями так не работает, но объявлять можно сколько угодно раз.
Но определить (фигурными скобками) можно только 1 раз (redeclaration).
Инициализация бывает только для переменных.
Для функций бывают определения и объявления.

```cpp
void f(int x); // объявление
void f(int x); // объявление
void f(int x) {};  // определение
```

Определять class можно в пределелах одного `translation unit`(единица трансляции) только один раз, но во всей программе сколько угодно раз, но при условии, что все определения дословно совпадают.

Любое определение являктся объявлением.


## Перегрузка функций

```cpp
void f();
void g() { f(); }
void f() { g(); }

// function overloading
int f(int x) { return x + 1; }
int f(double x) { return x + 2; }
```

`Promotion` считается более приоритетным, чем `convertion`:
```cpp
int f(int x) { return x + 1; }
int f(double x) { return x + 2; }

int main() {
	std::cout << f(0.0f); // выберется версия с double
}
```

но здесь компилятор не может решить, для него int и float равнозначны (и то, и другое является стандартной конверсией)
```cpp
int f(int x) { return x + 1; }
int f(float x) { return x + 2; }
int main() {
	std::cout << f(0.0); // ошибка компиляции
}
```

`promotion` лучше `conversion`, 
`conversion` стандартный лучше, чем определенный пользователем

Неоднозначность выбора имени, обращения к функции - ошибка этапа компиляции.

ошибка компиляции (redeclaration):
```cpp
void f();
int f();
```


виды обращений (квалифицированный/не индификатор):
```cpp
x;  // unqualified id
N::x;  // qualified id
```

на `namespace` правило одного определения не распростроняется.
внутри `namespace` можно объявить и определить другой `namespace`


```cpp
int x;  // определил, но не инициализировал
int x = 5; // инициализировал
```
По стандарту, всякое объявление переменной является ее определением, кроме записи вида `extern int x`;
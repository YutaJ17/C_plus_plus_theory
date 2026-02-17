## Шаблоны

### Идея шаблонов. Базовые примеры

Мы хотим научиться писать обощенный код. 
Для любого типа `T` определим функцию `f`, которая `T` принимает в качестве некой _метапеременной_.

```cpp
#include <iostream>

template <typename T>
void swap(T& x, T& y) {
	T y = x;
	x = y;
	y = t;
}

template <typename T>
int max(T x, T y) {
	return x > y ? x : y;
}
```
`template` воспринимать как квантор (для любого/любых типа/типов)

*Шаблонный класс*:
```cpp
template <typename T>
class vector {
	T* arr;
	size_t sz;
	size_t cap;
};
```

*Шаблонный using* (`C++11`):
```cpp

template <typename T>
struct less {
	bool operator()(const T& x, const T& y) const {
		return x < y;
	}
};

template <typename T> 
using mymap = std::map<T, T, std::greater<T>>;
```
`std::greater` - класс стандартной библиотеки, являющийся сомпоратором (на больше сравнивает)

Начиная с `C++14` можно делать _шаблонные переменные_.
Начиная с `C++20` можно после `typename` объявлять _концепт_.

---

```cpp
template <typename T, typename U>
```

```cpp
#include <iostream>
#include <map>

template <typename T>
void swap(T& x, T& y) {
	T y = x;
	x = y;
	y = t;
}

template <typename T> 
using mymap = std::map<T, T, std::greater<T>>;
mymap<int> m;

int main() {
	int a = 0;
	long long b = 1;
	swap(a, b);
}
```
Если компилятор не понимает как интерпретировать шаблон, будет `CE`.
	
	Надо воспринимать шаблоны, как паттерн, по которому компилятор должен скомпилировать код.

```cpp
#include <iostream>

template <typename T>
void swap(T& x, T& y) {
	T y = x;
	x = y;
	y = t;
}

int main() {
	int a = 0;
	int aa = 11;
	long long b = 1;
	long long bb = 22;
	swap(a, aa);
	swap(b, bb);
	swap(a, b);  // CE
	
}
```
Для всех упоминаний `swap` компилятор должен понять, какой `T` имеется ввиду и сгенерировать соответствующую версию.

Но компилятору можно подсказать шаблонный аргумент:
```cpp
#include <iostream>

template <typename T>
void swap(T x, T y) {
	T y = x;
	x = y;
	y = t;
}

int main() {
	int a = 0;
	int aa = 11;
	long long b = 1;
	long long bb = 22;
	swap(a, aa);
	swap(b, bb);
	swap<long long>(a, b);   // все ok
	
}
```

```cpp
#include <vector>
#include <iostream>

int main() {
	std::vector<int> v;
	std::vector<double> v2 = v;  // CE
}
```

### Перегрузка шаблонных функций

```cpp
#include <iostream>

template <typename T>
void f(T x) {}
void f(int x) {}

int main() {
	int x = 0;
	f(x);
}
```
Вызов второй функции, или будет `CE`? Выберется вторая версия. 
Неформальное правило: частное лучше общего.

```cpp
#include <iostream>

template <typename T>
void f(T x) {
	stD::cout << 1;
}
void f(long long x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	f(x);  // 1
}
```


```cpp
#include <iostream>

template <typename T>
void f(T x) {
	stD::cout << 1;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	f<long long>(x);  // 1
}
```

```cpp
#include <iostream>

template <typename T>
void f(T x) {
	stD::cout << 1;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	f<int>(x);  // 1
}
```

Можно указывать шаблонные аргументы по умолчанию:
```cpp
#include <iostream>

template <typename T, typename U = int>
U f(T x) {
	stD::cout << 1;
	return 0;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	f<int>(x); 
}
```

---

```cpp
#include <iostream>

template <typename T, typename U  // CE
U f(T x) {
	stD::cout << 1;
	return x;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	int g = f<int>(x); 
}
```

```cpp
#include <iostream>

template <typename U, typename T> // все ok
U f(T x) {
	stD::cout << 1;
	return x;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	int g = f<int>(x); 
}
```

```cpp
#include <iostream>

template <typename U, typename T> 
U f(T x) {
	stD::cout << 1;
	return x;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	long long x = 0;
	int g = f<int>(x); 
}
```

```cpp
#include <iostream>

template <typename U, typename T> 
U f(T x) {
	stD::cout << 1;
	return x;
}
void f(int x) {
	std::cout << 2;
}

int main() {
	std::string x = 0;
	int g = f<int>(x); 
}
```

```cpp
#include <iostream>

template <typename T> 
void f(T& x) {
	stD::cout << 1;
	return x;
}

template <typename T> 
void f(T x) {
	std::cout << 2;
}

int main() {
	int x = 0;
	f(x); // CE 
}
```

```cpp
#include <iostream>

template <typename T> 
void f(T& x) {
	stD::cout << 1;
	return x;
}

template <typename T> 
void f(T x) {
	std::cout << 2;
}

int main() {
	f(1); // 2 
}
```

```cpp
#include <iostream>

template <typename T> 
void f(const T& x) {
	stD::cout << 1;
	return x;
}

template <typename T> 
void f(T x) {
	std::cout << 2;
}

int main() {
	f(1); // CE
}
```

```cpp
#include <iostream>

template <typename T> 
void f(const T& x) {
	stD::cout << 1;
	return x;
}

template <typename T> 
void f(T& x) {
	std::cout << 2;
}

int main() {
	f(1);  // 2  (не требует конверсии)
}
```

---
Представьте, что вы хотите определить класс, который умеет консруироваться от любого `T`.  И вы хотите определить такому классу коструктор от `T&`. Как тогда будет работать копирование такого класса? Ведь такой конструктор поглотит коструктор копирования.

### Специализации шаблонов

При реализации шаблонного класса хочется, чтобы при определенном `T` класс был не таким, как при остальных. Этого можно добиться путем специализации шаблона.

```cpp
#include <iostream>

template <typename T>
class vector {
	T* arr;
	size_t sz;
	size_t cap;
};

template <>
class vector<bool> {
	char* arr;
	size_t sz;
	size_t cap;
};

template <typename T, typename U>
struct S {};

template <typename W>
struct S<W, W> {};  // частичная специализация

template <typename W>
struct S<int, W> {};

int main() {
	S<int, int> s;  // CE - неоднозначное инстанцирование шаблона
}
```

```cpp

template <typename T, typename U>
struct S {};

template <typename W>
struct S<W, W> {};  // частичная специализация

template <typename W>
struct S<int, W> {};

template <>
struct S<int, int> {};
int main() {
	S<int, int> s;  // Cвсе ok
}
```


```cpp
template <typename T>
struct A {};

template <typename T>
struct A<T&> {};

template <typename T>
struct A<const T> {};

int main() {
	A<const int&> a;  // все ok, ссылка приоритетней константной
}
```

```cpp
template <typename T>
struct A {};

template <typename T>
struct A<T&> {
	A() = delete;
};

template <typename T>
struct A<const T> {
	A() = delete;
};

int main() {
	A<const int&> a;  
}
```
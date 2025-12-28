У каждой переменной есть какой-то адрес в памяти. Значит, можно спросить номер ячейки памяти, в которой она хранится.
С помощью унарного ампеснанда `&` можно узать адрес ячейки:
```cpp
#include <iostream>
int main() {
	int x = 0;
	std::cout << &x;
}
```

Тип выражения `&x` - это `int *`. Здесь `*` - это часть типа.
```cpp
int x = 0;
int* p = &x;
```

Унарная звездочка (оператор) `*` показывает, что лежит под ячейкой памяти.
```cpp
int x = 0;
int* p = &x;  // Pointer
std::cout << &x;  // &: T -> T*  
std::cout << *p;  // *: T* -> T
```

`*p` - dereference (разыменование)
`&x` - address taken (взятие адреса)

```cpp
int main() {
	int x = 0;
	int y = 1;
	
	int* p = &x;
	p + 1;  // +(T*, int) -> T*
}
```

Указатели на разные типы по разному воспринимают сложения с целыми числами.
```cpp
int x = 1;
double y = 1.5;
int* p = &x;
int* p1 = &y;
p + 1;  // шаг на sizeof(int) = 4
y - 1;  // шаг на sizeof(double) = 8
p++;
y--;
++p;
--y;
```

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
int* p = &v[0];
std::cout << *p;  // 1
std::cout << *++p;  // 2
```

Можно брать разность двух указателей, если они одного типа. 
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
std::cout << &v[0] - &v[4];  // -4
std::cout << &v[3] - &v[2];  // 1
```

Для указателей определены операторы сравнения.

Можно брать указатели на указатели:
```cpp
#include <iostream>

int main() {
	int a = 0;
	int* p = &a;
	std::cout << p << '\n';  // некоторое 16ричное число
	int** pp = &p;  // &: int* -> int**
	std::cout << pp << '\n';  // другое 16ричное число
	
	std::cout << *pp << '\n';  // то же самое, что и std::cout << p
	std::cout << *p ' ' << **pp << '\n';  // то же самое, что и std::cout <<  a << ' ' << a;
}
```

```cpp
int x = 0;
int* p = &x;
std::cout << sizeof(p);  // implementation-defined behavior, но будет считать, что равен 8 байтам
```

```cpp
#include <iostream>
int main() {
	int a = 0;
	int* p = &a;
	std::cout << *&p << ' ' << &*p << '\n';  // 0 0
}
```

Унарная `*` - оператор `lvalue`.
```cpp
*p = 1;
```

`&` - `rvalue`, но его аргумент должен быть `lvalue`.
```cpp
&++p;  // можно
&p++;  // нельзя
```

Пример с UB:
```cpp
#include <iostream>

int main() {
	int a = 1;
	int* p = &a;
	{
		int b = 2;
		p = &b;
	}
	std::cout << p << '/n';
	std::cout << *p << '\n';  // UB, most likely 2
	
	int c = 3, d = 4, e = 5, f = 6;
	std::cout << &c << ' ' << &d << ' ' << &e << ' ' << &f << '\n';  // адрес f совпал с адресом b, память переиспользовалась
	
	++*p; 
	
	std::cout << c ' ' << d << ' ' << e << ' ' << f << '\n';
	std::cout << *p << '\n';  // 7
	
}
```

Действия с указателями разных типов:
```cpp
#include <iostream>

int main() {
	int x = 0;
	double d = 3.14;
	
	std::cout << (&x < &d);  // CE
	
	int* p = &x;
	*++p;  // UB
}
```

`void*` - указатель на неизвестно что, используется, чтобы иметь дело с указателями без уточнения, на область памяти чего конкретно мы указываем.
```cpp
#include <iostream>

int main() {
	int x = 0;
	int y = 1;
	int *p = &x;
	
	void* vp = &y;

}
```

`nullptr` - ноль в мире указателей

Зачем нужны указатели? Это способ сделать так, чтобы функции имели возможность влиять на переменные, существующие вне этих функций.
```cpp
void swap1(int x, int y) {  // работает локально
	int t = x;
	x = y;
	y = t;
}

void swap(int* x, int* y) {  // работает как задумывалось
	int t = *x;
	*x = *y;
	*y = t;
}

int main() {
	int x = 0;
	int y = 1;
	std::swap(x, y);
	std::cout << x << ' ' << y << '\n'; 
}
```


```cpp
#include <cstdio>  // для printf и scanf
#include <iostream>

int main() {
	int x = 1;
	int y = 2;
	
	printf("%d", x);  // 1
	scanf("%d", &y);  // введем в консоль 5
	std::cout << x << ' ' << y << '\n';  // 1 5
}
```

Можно указатели приводить к целым числам и наоборот:
```cpp
int x = 0;
int* p = &x;
std::cout << (long long)p;

int z = 324321;
std::cout << (int*)z;
std::cout << *(double*)z;  // seg fault
```

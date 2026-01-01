```cpp
int main() {
	int a[] = {1, 2, 3, 4, 5};
	int b[5];
	int c[6] = {1, 2};
	int d[3] = {};
	int e[7]{1, 2};
	int f[2]{}; 
	
	a[2] = 10;
}
```

Массив можно использовать в качестве указателя:
```cpp
int main() {
	int a[5] = {1, 2, 3, 4, 5};
	std::cout << *a << ' ' << *(a + 3);  // array to pointer conversion
	a[2] == *(a + 2);  // одно и тоже
	std::cout << 2[a];  // Ok, equivalent to *(2 + a)
	
	int* p = a + 3;
	std::cout << p[-1];  // 3, equivalent to *(p - 1) 
}
```
Но массивам нельзя ничего присваивать, в отличие от указателей. Также нельзя инкрементировать массив. Также оператор `sizeof()` от массива и от указателя будет разным.

```cpp
void f(int* a) {
	std::cout << a[2];
}

void f(int a[3]) {
	std::cout << "!";
	// так нельзя перегружать, будет redefenition
}

void f(int a[3]);  // так можно, есть разница между определением и объявлением
```

Массив динамически:
```cpp
#include <iostream>
int main() {
	int* a = new int[100]{};
	for (int i = 0; i < 100; ++i) {
		std::cout << a[i] << ' ';
	}
	delete[] a;
}
```

Массивы переменной длины - размер определеяется в `RunTime`.
```cpp
#include <iostream>
int main() {
	int n;
	std::cin >> n;
	
	int a[n];
	for (int i = 0; i < n; i++) {
		a[i] = i;	
	}
}
```

Двумерные массивы
```cpp

void f(int**) {}
void f(int(*)[5]) {
	std::cout << "hi!";
}
void f(int* [5]) {}  // тоже самое, что и 1 функция

int main() {
	int a[5][5];  // двумерный массив
	int* b[5];  // Array of 5 pointers to int
	int (*c)[5];  // Pointer to array of 5 ints
	
	int arr[5];
	f(&arr); // hi!
}
```

C-строки (null-terminated strings)
```cpp
int main() {
	char* s = "lksdksdf";
	char s1[] = "ankdk";  // сам текст хранится в статической памяти
	const char* s = "sflslls";
	
	const char* ss = "abc\0efg";
	std::cout << ss;; // abc
}
```


Указатели на функции - указатель на начало области памяти машинного кода, в котором записана реализация функции. Это может быть нужно при передаче функции в функцию.
```cpp
#include <iostream>
#include <sort>

bool cmp(int x, int y) {
	return x > y;
}

int main() {
	int a[5] = {5, 3, 8, 2, 9};
	
	bool (*p)(int, int) = &cmp;  // Function to pointer convertion
	std::cout << (void*)p;
	
	std::sort(a, a + 5);  // 2 3 5 8 9
	std::sort(a, a + 5, &cmp);  // 9 8 5 3 2
	

}
```

```cpp
#include <iostream>

void f(int) {}
void f(double) {}

int main() {
	&f;  // ошибка
	void (*p)(int) = &f;  
	void (*p1)(double) = &f;
	std::cout << (void*)p << ' ' << (void*)p1 << '\n';
	
}
```

`Function overloading` происходит не только при вызове функции, но и при взятии адреса. 

Что если одна из функций неопределена?
```cpp
void f(int);
void f(double) {};
```
Если функция неопределена, значит у нее нет тела, значит в машинном коде нет места, где записан код этой функции. 
Поэтому при попытке взять адрес у 
```cpp
void (*p)(int) = &f;
```
будет ошибка линковки.

## Default arguments

У функции могут быть аргументы по умолчанию. 
```cpp
void point(int x = 3, int y = 4);
point(1, 2);  // calls point(1, 2)
point(1);  // calls point(1, 4)
point();  // calls point(3, 4)
```

## Variadic functions

Пришло из языка C, это функции, в которые можно передавать переменное число аргументов. 
```cpp
void simple_prinf(const char* fmt...)
```

Чтобы такую функцию определить и чтобы достать аргументы, которые в нее передали, нужно использовать специальные макросы из `cstdarg`:
1. `va_start`
2. `va_arg`
3. `va_copy`
4. `va_end`
5. `va_list`
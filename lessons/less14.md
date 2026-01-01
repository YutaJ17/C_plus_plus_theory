Бывают ситуации, когда конструктор по умолчанию сгенерировать не получается:
```cpp
struct C {
	int& r;
	const int c;
};
```

Можно так:
```cpp
int x = 0;
struct C {
	int& r = x;
	const int c = x;
};
```

```cpp
int x = 0;

struct C {
	int &r = x;
	
	C(int y) {
		r = y;
	}
};
```

```cpp
int x = 0;

struct C {
	int &r = x;
	
	C(int y): r(y) {
	}
};
```
Если в полях ссылки или константы, обязательно нужно использовать списки инициализации.

```cpp
struct C {
	const int& r;
	C(): r(5) {}
};

int main() {
	C c;
	std::cout << c.r; // UB 
}
```
Константные ссылки можно инициализоровать через `rvalue`, они продлевают время жизни этим объектам до тех пор, пока не закончится время жизни самой ссылки, но работает это только при объявлении локальных переменных в функции.

## Деструктор

Это функция, которая вызывается перед выходом из области видимости объекта. Если конструктор делал нетривиальные действия, например, захват ресурса, то и деструктор должен быть нетривиален. 
А в классе `Complex` из предыдущего урока деструктор не нужен. 


```cpp

class String {
	char* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
public:
	String() {} // default constructor
	String(size_t n, char c): arr(new char[n + 1]), sz(n), cap(n + 1) {
		memset(arr, c, n);
		// std::fill(arr, arr + sz, c);
		arr[sz] = '\0';
		
	}
	String(std::initializer_list<char> list)
			: arr(new char[list.size() + 1])
			, sz(list.size()) 
			, cap(sz + 1) {
		std::copy(list.begin(), list.end(), arr);	
		arr[sz] = '\0';
	}
	
	~String() {
		delete[] arr;
	}
			
};
```

Деструктор для каждого объекта должен быть вызван только 1 раз, если больше, то `UB`.

```cpp
class String {
	char* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
public:
	String() {} // default constructor
	String(size_t n, char c): arr(new char[n + 1]), sz(n), cap(n + 1) {
		memset(arr, c, n);
		arr[sz] = '\0';
		
	}
	String(std::initializer_list<char> list)
			: arr(new char[list.size() + 1])
			, sz(list.size()) 
			, cap(sz + 1) {
		std::copy(list.begin(), list.end(), arr);	
		arr[sz] = '\0';
	}
	
	~String() {
		delete[] arr;
	}
};

int main() {
	String s = {'a', 'b', 'c'};
	s.~String();  // error: double free
}
```

## Конструктор копирования
Позволяет сделать объект класса из другого объекта такого же класса.
```cpp
String(const String& other); // copy constructor
```

Если не определить свой конструктор копирования (в нетривиальных случаях, например для класса String), то компилятор сгенерирует свой дефолтный конструктор копирования, из-за этого будет ошибка.

## COW string

Такой способ реализации (ленивое копирование), когда конструктор копирования оставляется тривиальным, а реальное копирование делается при попытки изменить копию.

## конструктор копирования для строки

```cpp
class String {
	char* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
public:
	String() {} // default constructor
	String(size_t n, char c): arr(new char[n + 1]), sz(n), cap(n + 1) {
		memset(arr, c, n);
		arr[sz] = '\0';
		
	}
	String(std::initializer_list<char> list)
			: arr(new char[list.size() + 1])
			, sz(list.size()) 
			, cap(sz + 1) {
		std::copy(list.begin(), list.end(), arr);	
		arr[sz] = '\0';
	}
	
	String(const String& other): 
			arr(new char[other.cap])
			, sz(other.size)
			, cap(other.cap) {
		memcpy(arr, other.arr, sz + 1);
		// memmove(arr, other.arr, sz + 1);
			}
	
	~String() {
		delete[] arr;
	}
};

int main() {
	String s = {'a', 'b', 'c'};
	s.~String();  // error: double free
}
```

```cpp
String(String&) = default;  // (перегрузка между ссылкой и конст ссылкой)
```

## делегирующие конструкторы

```cpp
class String {
	char* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
private:
	String(size_t n): arr(new char[n + 1], sz(n), cap(n + 1)) {
		arr[sz] = '\0';
	}
public:
	String() {} // default constructor
	String(size_t n, char c): String(n) {  // делегировали
		memset(arr, c, n);
	}
	String(std::initializer_list<char> list)
			: String(list.size()) {  // делегировали
		std::copy(list.begin(), list.end(), arr);	 
	}
	
	String(const String& other): String(other.sz) {  // делегировали
		memcpy(arr, other.arr, sz + 1);
	}
	
	~String() {
		delete[] arr;
	}
};

int main() {
	String s5 = s5;  // конструктор копирования строки от себя самой, так не надо делать
}
```

Избежали копипасты кода.
При входе в тело конструктора поля уже должны быть проинициализированы.

Можно запретить вызов функции:
```
String() = delete;
```


Для класса String выше нужно написать оператор присваивания. У оператора присваивания должен быть возвращаемый тип - ссылка на стринг.

```cpp
class String {
	char* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
private:
	String(size_t n): arr(new char[n + 1], sz(n), cap(n + 1)) {
		arr[sz] = '\0';
	}
public:
	String() {} // default constructor
	String(size_t n, char c): String(n) {  
		memset(arr, c, n);
	}
	String(std::initializer_list<char> list)
			: String(list.size()) { 
		std::copy(list.begin(), list.end(), arr);	 
	}
	
	String(const String& other): String(other.sz) {  
		memcpy(arr, other.arr, sz + 1);
	}
	
	String& operator=(const String& other) { 
		delete[] arr;
		sz = other.sz;
		cap = other.cap;
		arr = new char[other.cap];
		memcpy(arr, other.arr, sz + 1);
		return *this;
	}
	
	~String() {
		delete[] arr;
	}
};

```
Оператор присваивание выше не учитывает присваивание самому себе.

Нужно вот так:
```cpp
String& operator=(const String& other) { 
	if (this == &other) return *this;
	delete[] arr;
	sz = other.sz;
	cap = other.cap;
	arr = new char[other.cap];
	memcpy(arr, other.arr, sz + 1);
	return *this;
}
```

Но это можно сделать проще и короче через идиому copy and swap.

```cpp
void  swap(String& other) {
	std::swap(arr, other.arr);
	std::swap(sz, other.sz);
	std::swap(cap, other.cap);
}

String& operator=(const String& other) {
	String copy = other;
	swap(copy);
	return *this;
}
```

еще короче:
```cpp
String& operator=(String other) {
	swap(other);
	return *this;
}
```

## Правило трех

Если в классе есть нетривиальный конструктор копирования, или нетривиальный оператор присваивания, или нетривиальный деструктор, то обязательно нужно, чтобы все три были написаны.
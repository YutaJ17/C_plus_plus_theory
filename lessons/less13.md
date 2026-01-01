## Друзья

Друзья - такие функции или классы, которые хоть и не являются членами данного класса, им разрешен доступ к `private` полям.

```cpp
class C {
private:
	int x{5};
public: 
	void f(int y) {
		std::cout << x + y;
	}
	
	friend void g(C, int);  // объявление в любом месте класса C
	
};

void g(C c, int y) {
	std::cout << c.x + y + 1;
}

int main() {
	C c;
	std::cout << (int&)c;
}
```

Слово `friend` примерно как `reinterpret_cast`, нужно использовать только в крайнем случае. Если у класса должно быть много друзей, вероятно класс плохо спроектирован. 

```cpp
class C {
private: 
	class Inner {
		public:
			int x = 1;
		private:
			int y = 2;
	};

public:
	Inner f() {
		return Inner();
	}
};

int main() {
	C c;
	std::cout << c.f().x;  // CE или 1?
}
```
Ответ: 1.

```cpp
class C {
private:
	void f(int) {
		std::cout << 1;
	}
public:
	void f(float) {
		std::cout << 2;
	}	
};

int main() {
	C c;
	c.f(0);  // ?
	c.f(3.14);  // ?
}
```
в обоих случаях будет `CE`
В первом случае из-за ошибки прав доступа, во втором из-за `ambiguous call`.


## Конструкторы 

```cpp
class Complex {
	double re = 0.0;
	double im = 0.0;
public:
	Complex(double real) {
		re = real;
	}
};

int main() {
	Complex c(5.0);  // direct initialization
	Complex c2 = 6.0;  // value initialization
	Complex c3{7.0};  
	Complex c4 = {8.0};	
}
```

```cpp
class Complex {
	double re = 0.0;
	double im = 0.0;
public:
	Complex(double re) {
		this->re = re;  // плохой codestyle
	}
};
```

```cpp
class Complex {
	double re = 0.0;  // эта запись на случай, если не сказать, чем 
	double im = 0.0;  // проинициализировать
public:
	Complex(double re): re(real) {  // инициализация перед входом в конструктор (member initializer lists)
	}
};
```
Разница: сделать иницализацию 0, потом присваивание, или сразу сделать присваивание чем надо. Когда поля много весят и сам их конструктор не тривиален, это играет большую роль.

```cpp
class Complex {
	double re = 0.0;  
	double im = 0.0;  
public:
	Complex(double re);
	Complex(double re, double im): re(re), im(im) {}
};

Complex::Complex(double re): re(re) {}
```

Нарушение порядка инициализации полей:
```cpp
class Complex {
	double re = 0.0;  
	double im = 0.0;  
public:
	Complex(double re);
	Complex(double re, double im): im(im), re(re) {}
};
```

Если есть хоть один конструктор, то агрегатная инициализация перестает работать, и фигурные скобки будут означать вызов конструктора.

---

`Initializer lists` (не путать с `member initializer lists`)
```cpp
int main() {
	std::vector<int> v = {1, 2, 3, 4, 5};
}
```

В `vector` есть конструктор от `std::initializer_list`

Фигурные скобки `{}` могут означать:
1. агрегатную инициализацию
2. вызов конструктора
3. initializer list

```cpp
class Vector {
	int* arr = nullptr;
	size_t sz = 0;
	size_t cap = 0;
	
public:
	Vector() {} // default constructor
};

int main() {
	Vector v;  // default initialization
}
```
Если для примитивных типов написать `default constructor` и обращаться к ним, то это `UB`. Для классового типа это не `UB`.

```cpp
#include <cstring>

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
			
};

int main() {
	String s = {'a', 'b', 'c', 'd'};
	String s2{2, 'a'};
	String s3(2, 'a');  // обычный конструктор
}
```
Если есть хоть 1 конструктор от `std::initializer_list`, то далее при создании объекта первоочередно рассматриваются конструкторы от `std::initializer_list`, и если ни один из них не подходит, рассматриваются остальные.
Если создание объекта требует лишь инициализации полей, то тело конструктора и возможно сам конструктор не требуется.
`implicitly declared default constructor` - конструктор по умолчанию, который компилятор объявляет автоматически, если никаких других конструкторов не объявлено.

`expilitly declared, implicitrly defined default constructor` - явно объявленный, но неявно определенный 
```cpp
String() = default;
```
Иницилизирует все поля всем, чем нужно. 
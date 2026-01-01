```cpp
struct Complex {
	double re = 0.0;
	double im = 0.0;
	Complex(double re): re(re) {}
	Complex(double re, double im): re(re), im(im) {}
	
	Complex operator+(const Complex& other) const {
		return Complex{re + other.re, im + other.im};
	}
};

int main() {
	Complex c(1.0);
	c + 3.14;  // c.operator+(3.14);
	3.14 + c;  // CE
}
```
Стоит определять такие операторы вне класса, иначе они будут нессиметричны (пример выше). Но при опрелении вне класса такого оператора нужно прописывать 2 аргумента, внутри класса первый аргумент (объект класса) задан неявно.

```cpp
struct Complex {
	double re = 0.0;
	double im = 0.0;
	Compex(double re): re(re) {}
	Complex(double re, double im): re(re), im(im) {}
	
	Complex& oprator+=(const Complex& other) {  
		re += other.re;
		im += other.im;
		return *this;
	}
};

Complex operator+(const Complex& a, const Complex& b) {
	Compex result = a;
	result += b;
	return result;
}

int main() {
	Complex a(1.0);
	Complex b(2.0);
	Complex c(3.0);
	
	a + b = c;  // CE or OK?
}
```
OK, так как присваивание у кастомного типа к `rvalue` не запрещено.
Чтобы запретить, нужно записать вот так:
```cpp
Complex& operator=(Complex& other) & {}
```
Если `&&`, то только к `rvalue`.

---

```cpp
std::ostream& operator<<(std::ostream& out, const String& str);
std::istream& operator>>(std::istream& out, String& str);
```

```cpp
struct Complex {
	double re = 0.0;
	double im = 0.0;
	Compex(double re): re(re) {}
	Complex(double re, double im): re(re), im(im) {}
	
	Complex& oprator+=(const Complex& other) {  
		re += other.re;
		im += other.im;
		return *this;
	}
};

Complex operator+(const Complex& a, const Complex& b) {
	Compex result = a;
	result += b;
	return result;
}

bool operator<(const Complex& a, const Complex& b) {
	return a.re < b.re || a.re == b.re && a.im < b.im;
}

bool operator>(const Complex& a, const Complex& b) {
	return b < a;
}
```

## оператор spaceship

```cpp
struct Complex {
	double re = 0.0;
	double im = 0.0;
	Compex(double re): re(re) {}
	Complex(double re, double im): re(re), im(im) {}
	
	// Three-way comparison (since C++20)
	std::weak_ordering operator <=>(const Complex& other) = default;
};
```

## перегрузка инкремента

```cpp
struct UserId {
	int value = 0;
	
	UserId& operator++() {
		++value;
		return *this;
	}
	
	UserId operator++(int) {
		UserId copy = *this;
		++value;
		return copy;
	}
};
```

## перегрузка ()

```cpp
class Greater {  // функтор
	bool operator()(int x, int y) const {
		return x > y;
	}
};

int main() {
	std::vector<int> v(10);
	//...
	std::sort(v.begin(), v.end(), Greater());
}
```
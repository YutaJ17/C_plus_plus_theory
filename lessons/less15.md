```cpp
struct S {
	void f() {
		std::cout << "hi!";
	}
};

int main() {
	const S s;
	s.f();  // CE - ошибка нарушения константности
}
```

```cpp
struct S {
	void f() const {
		std::cout << "hi!";
	}
};

int main() {
	const S s;
	s.f();  
}
```

В константных методах можно вызывать только константные операции над полями.

```cpp
struct S {
	char arr[10];

	int x = 0;
	void f() const {
		std::cout << 1;
	}
	void f() {
		std::cout << 2;
	}
	
	char& operator[](size_t index) {
		return arr[index];
	}
	
	const char& operator[](size_t index) const {
		return arr[index];
	}
};

int main() {
	S s;
	const S& r = s;
	r.f();
}
```


```cpp
x = 0;

struct S {
	char* const arr;
	int& r = x;
	void f() const {
		std::cout << 1;
	}
	void f() {
		std::cout << 2;
	}
	
	char& operator[](size_t index) {
		return arr[index];
	}
	
	const char& operator[](size_t index) const {
		return arr[index];
	}
};
```

`mutable` - анти конст, нужен, когда объект константный с точки зрения пользователя, но в реализации он делает какие-то внутренние перестроения

```cpp
struct S {
	mutable int x = 1;
	int& r = x;
};
```

`static` - относятся к классу в целом, а не к объекту. То есть можно пользоваться без создания объекта
```cpp
struct S {
	static void f() {
		std::cout << "Hi";
	}
};

int main() {
	S::f();
}
```

`static` могут быть поля:
```cpp
struct S {
	static int x;
};
int S::x = 1;
```
Переменная x будет лежать в статической памяти.

```cpp
struct S {
	static const int x = 1;
}
```

Классический пример применения статических методов и функций  - класс `Singleton`. Это класс, который существует в единственном экземпляре.

```cpp
class Singleton {
	private:
		Singleton() {}
		static Singleton* ptr; // это не считается определением
		Singleton(const Singleton&) = delete;
		Singleton& operator=(const Singleton&) = delete;
	public:
		static Singleton& getObject() {
			if (ptr == nullptr) {
				ptr = new Singleton();
			}
			return *ptr;
		}
};

Singleton* Singleton::ptr = nullptr;

int main() {
	Singleton& s = Singleton::getObject();
}
```
Статическое поле нельзя проинициализировать внутри класса.

`explicit` (явный) — ключевое слово, применяемое к конструкторам и операторам преобразования типов, чтобы запретить неявное преобразование или неявный вызов этих методов компилятором.

```cpp
struct Latitude {
	double value;
	explicit Latitude(double value): value(value) {}
	
	explicit operator double() const {  // оператор приведения типа
		return value;
	}
}

struct Longitude {
	double value;
	explicit Longitude(double value): value(value) {}
}
```
То есть, если кто-то ожидает тип `Latitude`, а вы отдаете тип `double`, то не будет рассматриваться вариант неявной конверсии из `double` в `Latitude`.


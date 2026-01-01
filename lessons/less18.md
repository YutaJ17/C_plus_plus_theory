## видимость и доступы

```cpp
struct Base {
	int x;
	void f(int) {
		std::cout << 1;
	}
};

struct Derived: Base {
	int y;
	void f(double) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0); 
}
```
У наследника присутствует метод f, определенный у Base. Он скрыт. Доступен, но невидим. Что обратиться к нему явно, нужно использовать `qualified id`:

```cpp
struct Base {
	int x;
	void f(int) {
		std::cout << 1;
	}
};

struct Derived: Base {
	int y;
	void f(double) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.Base::f(0); // 1
}
```

3 этапа:
1. Из области видимости берутся кандидаты на перегрузку
2. Выбираем, кто "победил" в перегрузке
3. Проверяем приватность

Можно внести имена из одной области видимости в другую:
```cpp
struct Base {
	int x;
	void f(int) {
		std::cout << 1;
	}
};

struct Derived: Base {
	using Base::f;
	int y;
	void f(int) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0); // 2
}
```

```cpp
struct Base {
	int x;
	void f(double) {
		std::cout << 1;
	}
};

struct Derived: Base {
	using Base::f;
	int y;
	void f(int) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0.0); // 1
}
```

```cpp
struct Base {
private:
	int x;
	void f(double) {
		std::cout << 1;
	}
};

struct Derived: Base {
	using Base::f;
	int y;
	void f(int) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0.0); // CE
}
```

```cpp
struct Base {
protected:
	int x;
	void f(double) {
		std::cout << 1;
	}
};

struct Derived: Base {
	using Base::f;
	int y;
	void f(int) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0.0); // 1
}
```

С помощью `using` можно то, что доступно только в наследнике, привнести во вне. Но при этом важна приватность `using`. 

```cpp
struct Base {
	int x;
	void f(double) {
		std::cout << 1;
	}
};

struct Derived: Base {
private:
	using Base::f;
public:
	int y;
	void f(int) {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	d.f(0.0); // CE
}
```

Видимость - это то, что попало в область видимости. Доступность - то что можно использовать с точки зрения прав доступа.  Сначала выбирается область видимости, потом делается перегрузка, потом проверяется доступ. 


С полями все проще тем, что у них нет перегрузки:
```cpp
struct Base {
	int x = 1;
};

struct Derived: Base {
	int x = 2;
};

int main() {
	Derived d;
	std::cout << d.x; 
}
```

Приватность совершенно не влияет на то, как в памяти все это располагается, какие методы есть, каких нет.

```cpp
struct Base {
	int x = 1;
};

struct Derived: Base {
	int x = 2;
};

int main() {
	Derived d;
	std::cout << d.Base::x; // 1
}
```
`d.Base::x` - смотри `x` в области `Base`

```cpp
struct Granny {
	int x;
	void f() {}
};

struct Mom : private Granny {
	friend int main();
	int x;
}

struct Son : Mom {
	int x;
};

int main() {
	Son s;
	s.Mom::Grany::x;
	s.Granny::x;
	s.Mom::x;
}
```

что в этом коде может пойти не так?
```cpp
struct Granny {
	int x;
	void f() {}
};

struct Mom : private Granny {
	friend struct Son;
	friend int main();
	int x;
}

struct Son : Mom {
	int x;
	void f(Granny& g) { 
		std::cout << g.x;
	}
};

int main() {
	Son s;
	s.Granny::x;
}
```
В контексте сына, Granny - приватное имя, и даже принять его по ссылке нельзя. Чтобы это работало, нужно написать:

```cpp
struct Granny {
	int x;
	void f() {}
};

struct Mom : private Granny {
	friend struct Son;
	friend int main();
	int x;
}

struct Son : Mom {
	int x;
	void f(::Granny& g) {   // здесь добавили ::
		std::cout << g.x;
	}
};

int main() {
	Son s;
	s.Granny::x;
}
```

## memory layout in case of inheritance 

Как размещаются в памяти объекты родителей и наследников?
```cpp
#include <iostream>

```

Интуиция: если вы наследуетесь от чего-то, оно становится частью вас. Все его поля становятся вашими полями, просто они еще заключены во внутренней оболочке (Base).

```cpp
struct Base {
	int x;
};

struct Derived : Base {
	double y;
};

int main() {
	std::cout << sizeof(Derived);  // 16
}
```

```cpp
struct Base {
	void f() {}
};

struct Derived : Base {
	double y;
	void g() {}
};

int main() {
	std::cout << sizeof(Derived);  // 8
}
```
Наличие методом не влияет на размер (если только это не виртуальные методы).
Пустая структура занимает в памяти 1 байт.
Но в примере выше выведет 8 по правилу `EBO: Empty Base Optimization`. 

```cpp
struct Base {
	void f() {}
	Base(int x): x(x) {}
};

struct Derived : Base {
	double y;
	void g() {}
	Derived(double y): y(y) {}
};

int main() {
	Derived d = 3.14;
}
```
Когда создается объект наследника, сначала создается объект родителя. Чтобы код выше компилировался, нужно добавить:

```cpp
struct Base {
	void f() {}
	Base() = default;
	Base(int x): x(x) {}
};

struct Derived : Base {
	double y;
	void g() {}
	Derived(double y): y(y) {}
};

int main() {
	Derived d = 3.14;
}
```

Или так (дефолтный конструктор будет сгенерирован по умолчанию):
```cpp
struct Base {
	void f() {}
};

struct Derived : Base {
	double y;
	void g() {}
	Derived(double y): y(y) {}
};

int main() {
	Derived d = 3.14;
}
```

Или можно в конструкторе наследника указать, чем инициализировать родителя:
```cpp
struct Base {
	void f() {}
	Base(int x): x(x) {}
};

struct Derived : Base {
	double y;
	void g() {}
	Derived(double y): Base(0), y(y) {}  // строго в таком порядке
};

int main() {
	Derived d = 3.14;
}
```

```cpp
#include <iostream>

struct A {
	A(int x) { std::cout << "A" << x; }
};

struct Base {
	A x;
	Base(int x): x(x) { std::cout << "Base"; }
};

struct Derived : Base {
	A y;
	Derived(int y): Base(0), y(y) { std::cout << "Derived" << y; }
};

int main() {
	Derived d = 1;  // A0BaseA1Derived1
}
```
Сначала создаются поля родителя, вызывается тело конструктора родителя, далее - поля наследника, далее - вызывается тело конструктора наследника.

```cpp
#include <iostream>

struct A {
	A(int x) { std::cout << "A" << x; }
	~A() { std::cout << "~A" }
};

struct Base {
	A x;
	Base(int x): x(x) { std::cout << "Base"; }
	~Base() { std::cout << "~Base"; }
};

struct Derived : Base {
	A y;
	Derived(int y): Base(0), y(y) { std::cout << "Derived" << y; }
	~Derived() { std::cout << "~Derived"; }
};

int main() {
	Derived d = 1;  // A0BaseA1Derived1
	std::cout << "\n";
}
```
Из деструктора корректно обращаться к своим полям, к родителям и их методам, их время жизни еще не закончилось. Все происходит в порядке, обратном работе констуктора. Деструктор уничтожает поля.

Выведет после выхода из `main()`:
```
~Derived~A~Base~A
```

В `C++11` добавили наследование конструкторов:
```cpp
struct A {
	A(int x) { std::cout << "A" << x; }
	~A() { std::cout << "~A" }
};

struct Base {
	A x;
	Base(int x): x(x) { std::cout << "Base"; }
	~Base() { std::cout << "~Base"; }
};

struct Derived : Base {
	int y = 0;
	using Base::Base;
	Derived (int x, int y): Base(0), y(y) {}
};

int main() {
	Derived d = 1;
}
```

А что если у родителя был нетривиальный конструктор копирования:
```cpp
struct A {
	A(int x) { std::cout << "A" << x; }
	~A() { std::cout << "~A" }
};

struct Base {
	A x;
	Base(int x): x(x) { std::cout << "Base"; }
	Base(const Base& other): x(other.x) {std::cout <<"copy"; }
};

struct Derived : Base {
	int y = 0;
	using Base::Base;
	Derived (int x, int y): Base(0), y(y) {}
};

int main() {
	Derived d = 1;
	Derived d2 = d;  // от ссылки на Derived к ссылке на Base разрешен неявный каст
	std::cout << d2.y; // 1
}
```
У `Derived` неявно сгенерировался свой собственный конструктор копирования от `const Derived&`, и он побеждает в перегрузке между `const Derived&` и `const Base&`.

```cpp
struct A {
	A(int x) { std::cout << "A" << x; }
	~A() { std::cout << "~A" }
};

struct Base {
	A x;
	Base(int x): x(x) { std::cout << "Base"; }
	Base(const Base& other): x(other.x) {std::cout <<"copy"; }
};

struct Derived : Base {
	int y = 0;
	using Base::Base;
	Derived (int x, int y): Base(0), y(y) {}
};

int main() {
	Base b = 1;
	Derived d2 = b;  // CE, потому что конструкторы копирования не наследуются
	std::cout << d2.y; 
}
```

## Приведение типов при наследовании

```cpp
#include <iostream>

struct Base {
	int x;
};

struct Derived : Base {
	int y;
};

void f(Base& b) {
	std::cout << b.x;
}

int main() {
	Derived d;  // объект типа Derived
	f(d);
}
```
Можно использовать наследника там, где ожидается родитель. Ведь наследник умеет все, что может родитель, и что-то еще. (Основная идея наследования).

По аналогии: когда функция принимает константную ссылку, то в нее можно отдать как константу, так и неконстанту. Потому что константа - это просто тип, у которого убрана часть операций. То же самое - Base - это тип, у которого убрана часть операций по сравнению с Derived.
Произойдет неявный каст от наследника к родителю. Это каст того же сорта, как каст от `int&` к `const int&`.
В обратную сторону это не работает.

Можно ли принять Base по значению?
```cpp
#include <iostream>

struct Base {
	int x;
};

struct Derived : Base {
	int y;
};
// Slicing (срезка при копировании)
void f(Base b) {
	std::cout << b.x;
}

int main() {
	Derived d;  // объект типа Derived
	f(d);
}
```
`slicing`
Будет неявно сгенерирован конструктор Base от Derived, который заберет у Derived ту часть, которая относилась к Base.
Любой объект Derived содержит подобъект типа Base.
Понятно, что в обратную сторону нельзя. 

По указателю тоже можно:
```cpp
#include <iostream>

struct Base {
	int x;
};

struct Derived : Base {
	int y;
};

void f(Base* b) {
	std::cout << b->x;
}

int main() {
	Derived d;  
	f(d);
}
```

Еще немного про `slicing`. Представим, что `Base` - нетривиальный конструктор копирования:
```cpp
#include <iostream>

struct Base {
    int x = 1;
    Base() = default;
    Base(const Base& other): x(other.x) { std::cout << "Copy"; }
};

struct Derived : Base {
    int y = 2;
};
  
void f(Base b) {
    std::cout << b.x;
}

int main() {
    Derived d;  
    f(d);  // Copy1
}
```
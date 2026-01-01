## Указатели на члены

```cpp
struct S {
	int x;
	double y;
};

int main() {
	int S::* p = &S::x;
	S s{1, 3.14};
	s.*p = 2;
	std::cout << s.*p;
	
	S* ps = &s;
	std::cout << ps->*p;
}
```

```cpp
struct S {
	int x;
	double y;
	
	void f(int z) {
		std::cout << x + z;
	}
}

int main() {
	S s{1, 3.14};
	void (S::* pf)(int) = &S::f;  // указатель на метод
	(S.*pf)(3);
	(ps->*pf)(5);
}
```

## Енумы

Часто бывает нужно создать тип с несколькими именованными константами.

```cpp
enum E {
	White, 
	Gray, 
	Black
};

int main() {
	E e = White;
	// int e1 = Gray;
}
```

`enum` хранится в памяти как `int`.

`enum class` since C++11, они не вносят константы в глобальную область видимости и запрещают неявные конверсии в обе стороны.

```cpp
enum class E {
	White,
	Gray, 
	Black
};

int main() {
	E e = E::white;
}
```

Можно делать `static_cast` от `enum` к `int`.
Также можно:
```cpp
enum class E {
	White = 2,
	Gray = 5,
	Black = 1
}
```

```cpp
enum class E : int8_t {
	White = 1, 
	Gray,
	Black
}
```

## Наследование (`inheritance`)

```cpp
struct Base {
protected:
	int x;
public:
	void f() {}
};

struct Derived: Base {
	int y;
	void g() {
		std::cout << x;
	}
}

int main() {
	Derived d;
}
```

`protected` члены класса доступны другим членам класса, друзьям и наследникам.
Надо отличать понятия "доступны" и "видны".
Из области видимости выбирается имя, а после этого компилятор решает, доступно ли это имя.

К примеру выше, еще одна разница между структурой и классом:
```cpp
struct Derived: Base {};  // Base по умолчанию публичный родитель
class Derived: Base {};  // Base по умолчанию приватный родитель
```

```cpp
struct Derived: public Base {};
```
Публичный родитель - любая внешняя функция знает о том, что Base ваш родитель и имеет доступ ко всем полям и методам, которые вы унаследовали от него.

```cpp
struct Derived: private Base {};
```
Приватный родитель - только я и мои `friends` "знают" о том, что я - наследник. Никто другой знают и не могут использовать, что я - наследник Base.

```cpp
struct Base {
	int x;
};

struct Derived : private Base {};

int main() {
	Derived d;
	d.x;  // CE 
}
```

Защищенное наследование - только лишь я сам, мои друзья и наследники могут иметь доступ к моей родительской части.

```cpp
struct Granny {
	int x;
	void f() {}
};

struct Mom: protected Granny {
	int y;
	void g() {
		std::cout << x;
	}
};

struct Son : Mom {
	int z;
	void h() {
		std::cout << x;  // а вот так можно
	}
}

int main() {
	Son s;
	s.x; // CE
}
```

Как работает дружба при наследовании?

```cpp
struct Granny {
	friend int main();
	friend struct Son;
private:
	int x;
	void f() {}
};

struct Mom: protected Granny {
	int y;
	void g() {
		std::cout << x;  // CE
	}
};

struct Son : private Mom {
	int z;
	void h() {
		std::cout << x; // CE
	}
}

int main() {
	Son s;
	s.x; // CE 
}
```

```cpp
struct Granny {
	friend int main();
private:
	int x;
	void f() {}
};

struct Mom: private Granny {
	friend struct Son;
	int y;
	void g() {
		std::cout << x;  // CE
	}
};

struct Son : private Mom {
	int z;
	void h() {
		std::cout << x; // CE
	}
}

int main() {
	Son s;
	s.x; // CE 
}
```

```cpp
struct Granny {
	friend int main();
protected:
	int x;
	void f() {}
};

struct Mom: private Granny {
	friend struct Son;
	int y;
	void g() {
		std::cout << x;  // Ok
	}
};

struct Son : private Mom {
	int z;
	void h() {
		std::cout << x; // Ok
	}
}

int main() {
	Son s;
	s.x; // CE 
	
	Granny g;
	g.x;  // Ok
}
```

```cpp
struct Granny {
	friend int main();
protected:
	int x;
	void f() {}
};

struct Mom: private Granny {
	friend struct Son;
	int y;
	void g() {
		std::cout << x;  
	}
};

struct Son : private Mom {
	int z;
	void h(Granny& g) {
		std::cout << g.x;  // CE 
	}
}

int main() {

}
```

видимость и доступ

```cpp
struct Base {
	int x;
	void f() {
		std::cout << 1;
	}
};

struct Derived: Base {
	int y;
	void f() {
		std::cout << 2;
	}
}

int main() {
	Derived d;
	d.f();  // 2, частное предпочтительней общего
}
```

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
	d.f(0);  // 2, происходит затмение имен (другая функция в другой области видимости)
}
```

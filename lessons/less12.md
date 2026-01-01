## ООП
Объектно-ориентированное программирование. Будем объявлять свои типы (классы). Данные, которые хранятся в классах, будут называться полями. Операции, определенные над данными классами будут занываться методами. А переменные таких типов будут называться объектами.

```cpp
class C;

struct S;
```

Классы и структуры отличаются лишь привантостью, об этом позже.

```cpp
struct S {
	int x;
	double d = 3.14;
};

struct S1 {
	int x = 1;
}

int main() {
	S s;
	S1 s1;
	s1.x = 2;
	std::cout << s.x;  // UB
	std::cout << s1.x;
}
```

Размер структуры - сумма размеров ее полей с точностью до выравнивания.

```cpp
struct S {
	int x;
	double d = 3.14;
}; 

int main() {
	S s{2, 4.5};  // Aggregate initialization
	s.x = 3;
}

```

```cpp
struct S {
	int x;
	double d;

	void f(int y) {
		std::cout << x + y;
	}
};

int main() {
	S s{2};
	s.x = 3;
	s.f(5);  // 8
}
```

В классах и структурах можно пользоваться методами до момента их объявления. Сами методы определять вне класса.
```cpp
struct S {
	int x;
	double d;

	void f(int y);
	void ff() {}
};

void S::f(int y) {
	std::cout << x + y;
	ff();
}
```

Методы неявно первым аргументом принимают объект какого-то класса. Для этого есть ключевое слово `this`. Но это только указатель, чтобы получить объект, нужно написать `*this`.

```cpp
struct S {
	int x;
	double d;

	void f(int y);
	void ff(int x) {
		(*this).x;  // this->x; или так
	}
};

void S::f(int y) {
	std::cout << x + y;
	ff(y);
}
```

Внутри классов можно объявлять другие классы:
```cpp
struct A {
	int x;
	double d;
	
	// Inner class
	struct AA {  // сейчас размер A все еще 16 байт
		char c;
	} a;  // можно объявить переменную типа AA
	
	struct {  // без имени
		int r;
	} f;
};

int main() {
	A s;
	A::AA a;
	
	// Local class
	struct S {
		int x = 1;
		int y = 2;
	};
	
	S s{7, 10};
}
```

## Модификаторы доступа (`access modifiers`)


```cpp
class C {
	int x;
	int y;
};

int main() {
	C c;
	c.x;  // CE (ошибка с нарушением доступа)
}

```
 У класса по умолчанию `private` поля, у структуры `public`.

  ```cpp
class C {
public:
	int x;
private:
	int y;
protected:
	int z;
  };
  ```
К `private` полям можно обращаться только в самом классе, к `public` полям - откуда угодно, к `protected` - из самого класса и тех, кто от него наследуется.

Как достать значение приватного поля, обойдя защиту компилятора? Можно использовать `reinterpret_cast` или `C-style cast`. 
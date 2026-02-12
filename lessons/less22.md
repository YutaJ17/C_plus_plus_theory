## Memory layout of polymorphic objects with multiple inheritance

```
      Granny           Granny
        ↑                ↑
        |                |
       Mom              Dad
        ↑                ↑
         \              /
          ──────┬──────
                │
               Son
```
_Ромбовидное наследование: два экземпляра Granny_

```cpp
struct Granny { virtual void f(); int g; };
struct Mom : Granny { int m; };
struct Dad : Granny { int d; };
struct Son : Mom, Dad { int s; };
```

#### Расположение в памяти объекта Son:

| Смещение | Содержимое              | Принадлежность                                              |
| -------- | ----------------------- | ----------------------------------------------------------- |
| 0        | **vptr**                | Указатель на vtable для подобъекта <br>`Son → Mom → Granny` |
| 4/8      | `int g`                 | Granny (копия от Mom)                                       |
| 8/12     | `int m`                 | Mom                                                         |
| 12/16    | **vptr**                | Указатель на vtable для подобъекта <br>`Son → Dad → Granny` |
| 16/20    | `int g`                 | Granny (копия от Dad)                                       |
| 20/24    | `int d`                 | Dad                                                         |
| 24/28    | `int s`                 | Son                                                         |
| ...      | (выравнивание, padding) |                                                             |

*Смещения даны примерно: 4/8 — первое число для 32-бит, второе для 64-бит*

```cpp
struct Granny { virtual void f(); int g; };
struct Mom : Granny { int m; };
struct Dad : Granny { int d; };
struct Son : Mom, Dad { int s; };

int main() {
	Son son;
	Dad& d = son;
	
	Mom& m = dynamic_cast<Mom&>(d);  // каст из &Dad в &Mom
}
```

- Из `vtable` `Dad`-части извлекается информация о полном объекте (`Son`)
- RTTI (Run-Time Type Information) говорит, что полный объект — `Son`
- У `Son` есть подобъект `Mom`, до которого можно добраться
- Вычисляется смещение от `Dad`-части до `Mom`-части (отрицательное)

Если вы у `Dad` пытаетесь вызвать метод `f`, то нужно как будто вызвать его просто у `Son`, но помнить дополнительно, что считать начало своего объекта на 16 байт правее. Так работает `non-virtual thunk`. Это вспомогательная функция, генерируемая компилятором для коррекции указателя `this` при вызове виртуальной функции через базовый класс, расположенный не в начале объекта. 
В таблице `Dad in Son` есть `top offset`, `Son type_info`, `&thunk`.
`thunk` в свою очередь делает сдвиг `this` на `top offset` и `call Son::f()`

#### Тот же пример, но наследование вируальное

```cpp
struct Granny { virtual void f(); int g; };
struct Mom : virtual Granny { int m; };  // виртуальное наследование
struct Dad : virtual Granny { int d; };  // виртуальное наследование
struct Son : Mom, Dad { int s; };
```

При появлении виртуального наследования, помимо 
- `top_offset`(как далеко мы от начала объекта) еще появляется 
- `virtual_offset`(как далеко от нас начало виртуального предка).
##### Память объекта Son
```text
┌─────────────────────────┐
│ [0]    vptr (Mom)       │ ──→ vtable for Mom (in Son)
├─────────────────────────┤
│ [8]    int m            │    Mom-часть
├─────────────────────────┤
│ [12]   vptr (Dad)       │ ──→ vtable for Dad (in Son)
├─────────────────────────┤
│ [20]   int d            │    Dad-часть
├─────────────────────────┤
│ [24]   int s            │    Son-часть
├─────────────────────────┤
│ [32]   vptr (Granny)    │ ──→ vtable for Granny (in Son)
├─────────────────────────┤
│ [40]   int g            │    Granny-часть (ОБЩАЯ!)
└─────────────────────────┘
```

##### `Vtable` для Mom in Son:
```text
┌─────────────────────────────────┐
│ -16 (top offset)               │ ← смещение до начала Son
├─────────────────────────────────┤
│ 16 (virtual offset to Granny)  │ ← смещение до виртуальной Granny
├─────────────────────────────────┤
│ &type_info for Son             │
├─────────────────────────────────┤
│ &thunk to Son::f()             │
└─────────────────────────────────┘
```

##### `Vtable` для Dad in Son:
```text
┌─────────────────────────┐
│ -12 (top offset)        │ ← Dad смещён на -12
├─────────────────────────┤
│ &type_info for Son      │
├─────────────────────────┤
│ &thunk to Son::f()      │ ← thunk сдвигает на -12
└─────────────────────────┘

```

##### `Vtable` для Granny in Son:
```text
┌─────────────────────────┐
│ -32 (top offset)        │ ← Granny смещена на -32
├─────────────────────────┤
│ &type_info for Son      │
├─────────────────────────┤
│ &Son::f()               │ ← прямой вызов (без thunk!)
└─────────────────────────┘

```

Компилятор не знает на этапе компиляции, где именно в памяти будет Granny — это зависит от конкретного производного класса. Поэтому смещение до виртуального базового класса хранится в `vtable`.

## Non obvious problems with virtual functions

```cpp
struct Base {
	int x;
	static virtual void f() {}
};

struct Derived : Base {
};

int main() {
	Derived d;
}
```
`CE`, не может быть `static` и `virtual` вместе.

```cpp
struct Base {
	int x;
	virtual void f();
};

struct Derived : Base {
};

int main() {
	Derived d;
}
```
Ошибка линкера. Виртуальные функции нельзя оставлять без определения, даже если вы их вызываете, кроме ситуаций, когда они `pure virtual functions`.

```cpp
struct Base {
	virtual void h() = 0;
	void f() {
		std::cout << "f";
		h();
	}
	Base() {
		std::cout << "Base";
		h();
	}
	virtual ~Base() = default;
};

struct Derived : Base {
	int x;
	void h() override {
		std::cout << "h" << x;
	}
	Derived(int x = 0): x(x) {
		std::cout << "Derived";
	}

};

int main() {
	Derived d;
}
```
`CE`, `UB`, `RunTime Error`, `ошибка линкера`? 
Правильный ответ: ошибка линкера, по причине того, что при вызове виртуальной функции из конструктора механизм виртуальных функций отключается. 

	P.S: правильный ответ зависит от компилятора, в стандарте написано, что будет  `UB`.

```cpp
struct Base {
	virtual void h() = 0;
	void f() {
		std::cout << "f";
		h();
	}
	Base() {
		std::cout << "Base";
		f();
	}
	virtual ~Base() = default;
};

struct Derived : Base {
	int x;
	void h() override {
		std::cout << "h" << x;
	}
	Derived(int x = 0): x(x) {
		std::cout << "Derived";
	}

};

int main() {
	Derived d;
}
```
`RunTime error: pure virtual method called`
В `vtable Base` по адресу `h` лежит `pointer` на некоторое специальное место, в котором написан специальный код (заглушка), выводящая в консоль этот текст с ошибкой, дальше вызывает функцию `std::terminate`, которая вызывает `std::abort`, из-за чего пишется `aborted (core dumped)`.

	P.S: правильный ответ зависит от компилятора, в стандарте написано, что будет  `UB`.

По мере создания объекта виртуальная таблица, `virtual ptr` претерпевает изменения. На самом деле, когда создается полиморфный объект, происходит еще один этап перед началом создания родителя, до инициализации его полей. Это инициализация `vptr`.
При создании _полиморфного объекта_ сначала должен создаться его родитель, и первым делом `vptr` инициализируется указателем на `vtable` родителя. После инициализируются поля родителя, после - тело конструктора родителя.
На момент нахождения в конструкторе родителя `vptr` указывает на `vtable` родителя.
После того, как отработал конструктор родителя, `vptr` переставляется на `vtable Derived`, после создаются поля `Derived` и выполняется тело конструктора  `Derived`.
Это сделано для того, чтобы случайно не попасть в метод предка, который еще не существует.
При срабатывании деструктора все действия происходят в обратном порядке.

А как это работает при виртуальном наследовании?
Для этого нужны `construction vtable` и `virtual table of tables`.

---

```cpp
struct Base {
	int a = 1; 
	virtual int h() = 0;
	void f() {
		std::cout << "f";
		h();
	}
	Base(): a(f()) {
		std::cout << "Base";
		f();
	}
	virtual ~Base() = default;
};

struct Derived : Base {
	int x;
	int h() override {
		std::cout << "h" << x;
	}
	Derived(int x = 0): x(x) {
		std::cout << "Derived";
	}

};

int main() {
	Derived d;
}
```
При вызове виртуальной функции в списке инициализации она также вызывается не по правилам виртуальных функций.

```cpp
struct Base {
	virtual void f(int x = 1) {
		std::cout << "Base " << x;
	}
};

struct Derived : Base {
	void f(int x = 2) override {
		std::cout << "Derived " << x;
	}
};

int main() {
	Derived d;
	Base& b = d;
	b.f();
}
```
Что выведет?: 
- Base 1
- Base 2
- Derived 1
- Derived 2

    Правильный ответ: Derived 1. Аргумент по умолчанию нужно подставить в `Compile time`. Аргумент по умолчанию подставился на основании типа объекта (Base&). А в Base указан x = 1.

#### финальный пример

```cpp
#include iostream

struct Mother {
	int x = 0;
	virtual void f() {
		std::cout << x;
	}
};

struct Father {
	int y = 1;
	virtual void g() {
		std::cout << y;
	}
};

struct Son : Mother, Father {
	int z = 2;
	void f() override {
		std::cout << z;
	}
	void g() override {
		std::cout << z;
	}
};

struct S {
	long long a;
	long long b;
};

int main() {
	void (Mother::*p)() = &Mother::f;
	
	Son son;
	Mother& m = son;
	(m.*p1)(); // 2
	
	void (Father::*p2)() = &Father::g;
	Father& f = son;
	(f.*p2)();  // 2	
	std::cout << sizeof(p1) << ' '; // 16
	S s1 = reinterpret_cast<S&>(p1);
	S s2 = reinterpret_cast<S&>(p2);
	std::cout << ' ' << s1.a << ' ' << s1.b << std::endl;  // 1 0
	std::cout << ' ' << s2.a << ' ' << s2.b << std::endl;  // 1 0
}
```

```cpp
#include iostream

struct Mother {
	int x = 0;
	virtual void f() {
		std::cout << x;
	}
};

struct Father {
	int y = 1;
	virtual void g() {
		std::cout << y;
	}
};

struct Son : Mother, Father {
	int z = 2;
	void f() override {
		std::cout << z;
	}
	void g() override {
		std::cout << z;
	}
};

struct S {
	long long a;
	long long b;
};

int main() {
	void (Mother::*p)() = &Mother::f;
	
	Son son;
	Mother& m = son;
	(m.*p1)(); // 2
	
	void (Son::*p2)() = &Father::g;
	Father& f = son;
	(son.*p2)();  // 2	
	std::cout << sizeof(p1) << ' '; // 16
	S s2 = reinterpret_cast<S&>(p2);
	std::cout << ' ' << s2.a << ' ' << s2.b << std::endl;  // 1 16
}
```
`1 16` 
второе число показывает сдвиг относительно начала объекта.

```cpp
#include iostream

struct Mother {
	int x = 0;
	virtual void f() {
		std::cout << x;
	}
};

struct Father {
	int y = 1;
	virtual void g() {
		std::cout << y;
	}
};

struct Son : Mother, Father {
	int z = 2;
	void f() override {
		std::cout << z;
	}
	void g() override {
		std::cout << z;
	}
};

struct S {
	long long a;
	long long b;
};

int main() {
	void (Son::*p)() = &Mother::f;
	
	Son son;
	Mother& m = son;
	(son.*p)(); // 2
	
	void (Son::*p2)() = &Son::g;
	Father& f = son;
	(son.*p2)();  // 2	
	std::cout << sizeof(p1) << ' '; // 16
	S s2 = reinterpret_cast<S&>(p);
	std::cout << ' ' << s2.a << ' ' << s2.b << std::endl;  // 9 0
}
```
`9 0` 
первое число показывает сдвиг относительно виртуальной табицы.
1 последним битом чтобы отличать виртуальный указатель от невертуальных. В `RunTime` компилятор понимает, что это не реальный адрес функции, а сдвиг относительно начала виртуальной таблицы.
Таким образом, не меняя размер указателя на метод можно вкодировать в него сдвиг относительно начала виртуальной таблицы вместо кодирования адреса настоящей функции. Так работают указатели на виртуальные методы изнутри.

---

P.S: более подробная инфа по memory layout с примерами будет добавлена позже (в самой лекции не приводились примеры употребления virtual table of tables и прочее)
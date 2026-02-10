```cpp
struct Shape {
	virtual double area() const = 0;
	virtual ~Shape() = default;  
};

double Shape::area() const {  // определили pure virtual функцию
	return 0.0;
}

struct Square: Shape {
	double a;
	Square(double a): a(a) {}
	double area() const override {
		return a * a;
	}
};

struct Circle: Shape {
	double r;
	Circle(double r): r(r) {}
	double area() const override {
		return 3.14 * r * r;
	}
}

int main() {

	

	Circle c(1.0);
	c.Shape::f();  // вызывается версия родителя (отменяется виртуальность)
	
	std::cout << c.Shape::area();

	std::vector<Shape*> v;
	v.push_back(new Square(1.0));
	v.push_back(new Circle(1.0));
	
	for (Shape* s: v) {
		std::cout << s->area() << std::endl;
	}
}
```

Несмотря на то, что дано определение `pure virtual` функции, класс `Shape` все еще абстрактный, все еще нельзя создавать его экземпляры, только у его наследников.
При этом если наследник не определил такую функцию, он сам считается абстрактным, и так далее.

Это откат на случай, если какой-то из наследников не справился реализовать функцию, некая заглушка.

## `dynamic_cast`,  RTTI

`dynamic_cast` - каст в `RunTime`.

```cpp
struct Base {
	int x;
	virtial void f() {}
	virtual ~Base() = default;
};

struct Derived: Base {
	int y;
	void f() override {}
};

int main() {
	Derived d;
	Base& b = d;
	dynamic_cast<Derived&>(b); // тип выражения - Derived&
	dynamic_cast<Derived&>(b).y;  // применимо поле y
	
	Derived* pd = dynamic_cast<Derived*>(&b);
	if (pd) {
		// do smth
	}
}
```

Работает только для типов с виртуальными функциями.
Наличие виртуальной функции у типа (полиморфность) означает, для него поддерживается какая-то специальная информация, по которой в `RunTime` понятно, что это на самом деле.
`virtual functions` и `dynamic_cast` - очень тесносвязанные понятия.
Если в классе нет виртуальной функции, то `dynamic_cast` делать нельзя, будет `CE`.
Полиморфность типа - необходимое условие для того, чтобы `dynamic_cast` мог проверить в `RunTime`, что это за тип на самом деле.
`dynamic_cast` - дорогая операция, нужно по указателю найти табличку, а по написанному в ней понять реальный тип, и уже в зависимости от этого уже понять, можно ли делать этот каст, сдвигать ли `ptr` или нет.

`RTTI` - run time type information. 
Для полиморфных типов компилятор поддерживает специальную структуру, в которой хранит, что это за тип на самом деле. То есть в каждом объекте полиморфного типа есть скрытый pointer, указывающий на нечто, где написано, что это на самом деле за тип.

Не нужно в своем типе заводить отдельные поля, в которых указан тип, для этого есть механизм `RTTI`.

```cpp
int main() {
	Derived d;
	Base& b = d;
	std::cout << typeid(b).name() << std::endl;  // оператор typeid
}
```
`typeid` работает не только для полиморфных типов.
Возвращает объект типа `std::type_info`, их можно сравнивать.

if с инициализацией:
```cpp
if (Derived* pd = dynamic_cast<Derived*>(&b); pd) {
	// 
}
```

От любого полиморфного типа можно делать `dynamic_cast` от указателя, а от неполиморфного - нельзя, таким образом можно проверить полиморфность типа.
Также можно всегда делать `dynamic_cast` вверх, от наследника к родителю (даже если тип неполиморфный). Но в этом случае (родитель -> наследник) такой каст нужно использовать, только если нужны проверки в `RunTime` (так или иначе идет работа с виртуальными функциями). Для проверки в `CT` используется `static_cast`.  Чем еще удобно: если ошибиться в касте от наследника к родителю (сделать наоборот), то, используя `static` будет `UB`, а, используя `dynamic` - `CE`.
Также `dynamic_cast` может кастовать "вбок" при множественном наследовании. При этом полиморфным должен быть тот, от кого кастятся.

Если нужно сделать тип полиморфным, а виртуальные функции не нужны, можно прописать виртуальный деструктор.

При касте вбок (от маме к папе):
- `static_cast` дает  `CE`
- `dynamic_cast` корректно отрабатывает
- `reinterpret_cast` дает `UB`

При касте вниз при виртуальном наследовании (предок полиморфный):
- `static_cast` дает  `CE`
- `dynamic_cast` корректно отрабатывает
- `reinterpret_cast` дает `UB`

Виртуальная функция - такая функция, что решение о том, какую функцию выбрать, принимается в  `RunTime`, потому что у полиморфных типов есть специальная информация `type_info`, по которой становится понятно, какую версию функции выбрать.

## Memory layout of polymorphic objects

Информация ниже - не совсем стандарт C++, а стандарт ABI - бинарный интерфейс (как компилятор по отношении к операционной системе и процессору должен в памяти распологать)

```cpp
#include <iostream>

struct Base {
    virtual void f() {}
};

int main() {
    std::cout << sizeof(Base) << std::endl;  // 8
}
```
В Base есть виртуальная функция, значит у него должен быть указатель на `vtable` (таблицу виртуальных функций).
Это структура данных, хранящаяся в статической памяти, одна на тип. В которой перечислены адреса виртуальных функций этого типа.

Что виртуальное наследование, что виртуальные функции разрешаются через одну и ту же структуру данных, называемую виртуальной таблицей. В ней хранятся как адреса виртуальных предков, так и адреса виртуальных методов.

В `vtable` для Base хранятся:
- `&Base::f`
- `&Base type_info`

```cpp
#include <iostream>

struct Base {
    int x;
    virtual void f() {}
    void h() {}
};

struct Derived : Base {
    void f() override {}
    virtual void g() {}
    int y;
};

```
В памяти `Derived` будет хранится следующим образом:
`ptr(указывает на vtable для Derived)`, `x`, `y`

`vtable для Derived`: `Derived type_info`, `&Derived::g` 

```cpp
struct Base {
    int x;
    virtual void f() {}
    void h() {}
};

struct Derived : Base {
    void f() override {}
    virtual void g() {}
    int y;
};

int main() {
	Base b;
	b.f();
}
```

При вызове `b.f()` происходит переход (в `RunTime`) по `vptr` у Base на `vtable` Base. Зная, что `f()` - первая из виртуальных функций объекта Base, компилятор отступает на 0 байт и считывает 8 байт, интерпретируя их как `pointer`. Этот `pointer` и будет указывать на функцию `f()`.

```cpp
struct Base {
    int x;
    virtual void f() {}
    void h() {}
};

struct Derived : Base {
    void f() override {}
    virtual void g() {}
    int y;
};

int main() {
	Derived d;
	Base& b = d;
	b.f();
}
```
В данном случае `vptr` будет вести на таблицу `Derived`, а не `Base`.

Как работает `dynamic_cast`? 
Он идет по `vptr`, находит `type_info` в  `vtable`,  понимая какой тип на самом деле. В зависимости от этого он либо кидает ошибку, либо сдвигает `ptr`.

```cpp
struct Granny {
	void h() {}
	int g;
};

struct Mom : Granny {
	virtual void f() {}
	int m;
};

struct Son : Mom {
	void f() override {}
	int s;
};
```
Как в памяти выглядит объект `Son`?
`vptr`, `g`, `m`, `s`
При касте от `Son` к  `Granny` нужно сдвинуть pointer, так как `Granny` неполиморфна. Но при этом от `Granny` к `Son` `dynamic_cast` невозможен, так как `source type is not polymorphic`.  

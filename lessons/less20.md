## Идея виртуальных функций

Важно не путать виртуальное наследование и виртуальные функции. Они являются родственными понятиями, но в пределах данной лекции их лучше таковыми не считать.

```cpp
struct Base {
	void f() {
		std::cout << 1;
	}
};

struct Derived : Base {
	void f() {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	Base& b = d;
	b.f();  // 1
}
```

Пример: есть поваренная книга с инструкциями, как варить крупы: рецепт для общего вилда круп и конкретные рецепты. Если пользователю дать рис и сказать, чтобы он сварил эту крупу, было бы славно, если бы он ее варил конкретному рецепту для риса. 
Другой пример: на плоскости есть 2 круга одинакового радиуса, но у них разные координаты центров. В общем случае эти два круга равны (их можно совместить друг на друга), но в частном, как равенство двух множеств точек, они неравны. 
Эта та идея, которая стоит за понятиями виртуальных и невиртуальных функций.

При желании сделать так, чтобы при использовании ссылки на `Base`, вызвался метод у `Derived`. Чтобы этого добиться, нужно написать `virtual` перед функцией:
```cpp
struct Base {
	virtual void f() {
		std::cout << 1;
	}
};

struct Derived : Base {
	void f() {
		std::cout << 2;
	}
};

int main() {
	Derived d;
	Base& b = d;
	b.f();  // 2
}
```
Теперь выведется 2.

_Виртуальная функция_ - это такая функция, что будучи вызванной у объекта по ссылке на родителя она все равно работает, как версия от наследника.
Также работает через указатель:
```cpp
int main() {
	Derived d;
	Base* b = &d;
	b->f();  // 2
}
```

А вот так:
```cpp
int main() {
	Derived d;
	Base b = d;
	b.f();  // 1
}
```
Выведется 1, так как здесь полноценный Base.

Тип, у которого есть хотя бы одна виртуальная функция, называется _полиморфным типом_. В том числе, если эта функция унаследована.

```cpp
struct Base {
	virtual void f() {
		std::cout << 1;
	}
};

struct Derived : Base {
	int* p = new int(0);
	void f() {
		std::cout << 2;
	}
	~Derived() {
		delete p;
	}
}

int main() {
	Base* b = new Derived();  // неявный каст (implicit cast) от указателя на родителя к указателю на наследника
	delete b;
}
```

Здесь происходит утечка памяти. Деструктор вызывается от типа переменной. То есть в примере выше деструктор вызвался для Base, а он тривиален.

Оператор `delete` вызывает деструктор и освобождает память.
Оператор `new` выделяет память и вызывает конструктор.

Чтобы это исправить, нужно в Base сделать виртуальный деструктор:
```cpp
struct Base {
	virtual void f() {
		std::cout << 1;
	}
	virtual ~Base() = default;
};

struct Derived : Base {
	int* p = new int(0);
	void f() {
		std::cout << 2;
	}
	~Derived() {
		delete p;
	}
}

int main() {
	Base* b = new Derived();  
	delete b;
}
```

## virtual examples

Неважно, писать `virtual` только в родителе или и родителе, и в наследнике. 

Виртуальной считается только та функция, которая в точности совпадает по сигнатуре.

```cpp
struct Base {
	virtual void f() const {
		std::cout << 1;
	}
};

struct Derived : Base {
	void f() {
		std::cout << 2;
	}
	
}

int main() {
	Derived d;
	Base& b = d;
	b.f();  // 1
}
```
Выведется 1, а не 2, так как не совпадает сигнатура в наследнике.

Писать у наследника `veritual` при переопределении виртуальной функции не обязательно, а вот что обязательно, так это писать слово `override`:

```cpp
struct Base {
	virtual void f() const {
		std::cout << 1;
	}
};

struct Derived : Base {
	void f() override {
		std::cout << 2;
	}
	
};

int main() {
	Derived d;
	Base& b = d;
	b.f();  // CE: marked ‘override’, but does not override
}
```
Это слово обозначает, что я хочу переопределить какую-то виртуальную функцию из одного из родителей. 
Зачем это нужно? Если написать `override`, и ни у одного из родителей нет виртуальной функции, то это будет `CE`. От наличия слова `override` не меняется поведение программы, только увеличивается количество ошибок компиляции.

Если при переопределении виртуальной функции сделать другой возвращаемый тип, но ту же сигнатуру, будет `CE`.

Еще для `virtual functions` можно написать `final`. 
Так можно запретить всем своим дальнейшим наследникам переопределять эту функцию (с такой же сигнатурой).

```cpp
void f() const final {
	std::cout << 2;
}
```

Более глубокий смысл `final` - это помогает компилятору оптимизировать код.

Из слов `override` и `final` нужно всегда максимум одно, так как `final` и так обозначает `override`.

Другая мысль:  из слов `virtual ... override` и `final` нужно максимум одно.
1. При определении новой виртуальной функции пишем `virtual`.
2. При переопределении уже существующей виртуальной функции пишем `override` и `virtual` из него автоматически следует. 
3. При доопределении последней виртуальной функции  пишем `final`, из которого следует `override`.

Смысл виртуальных функций в том, чтобы у родителей и наследников они вели себя по разному и в зависимости от того, через кого они вызвались, было бы разное поведение.

`override` и `final` - это `context sensitive keywords` (контекстно-зависимые ключевые слова)
```cpp
int override = 5;  // можно написать так, но смысла в этом не будет
```
Эти слова несут смысл только в определенном контексте.

Другой пример:
```cpp
struct Granny {
	virtual void f() const {
		std::cout << 1;
	}
};

struct Mom : Granny {
	void f() const override {
		std::cout << 2;
	}
};

struct Son : Mom {
	void f() const final {
		std::cout << 3;
	}
};

int main() {
	Mom m;
	Granny& g = m;
	g.f();  // 2
}
```

А если добавить приватность?
```cpp
struct Granny {
	virtual void f() const {
		std::cout << 1;
	}
};

struct Mom : Granny {
private:
	void f() const override {
		std::cout << 2;
	}
};

struct Son : Mom {
	void f() const final {
		std::cout << 3;
	}
};

int main() {
	Mom m;
	Granny& g = m;
	g.f();  // 2
}
```
Почему выведется 2, а не `CE`? Потому что виртуальные функции - это `RunTime` явление, а приватность - это `Compile time` явление. 
Компилятор в `Compile time` не может никак понять, попадете вы в публичную или в приватную функцию, это решается в `Run time`.

Пример, доказывающий, что в `compile time` это невозможно определить:
```cpp
struct Granny {
	virtual void f() const {
		std::cout << 1;
	}
};

struct Mom : Granny {
private:
	void f() const override {
		std::cout << 2;
	}
};

struct Son : Mom {
	void f() const final {
		std::cout << 3;
	}
};

int main() {
	Mom m;
	Granny g;
	
	int x;
	std::cin >> x;
	
	Granny& gg = (x % 2 ? m : g);
	
	gg.f();
}
```
Выбор версии виртуальной функции - это стадия некомпиляции, она идет после проверки на приватность. А в данной программе компилятор не может проверить приватность.
Ну а если `private` будет стоять у `Granny`, то это `CE`.

Другой пример:
```cpp
struct Mom {
	virtual void f() {
		std::cout << 1;
	}
};

struct Dad {
	void f() {
		std::cout << 2;
	}
};

struct Son : Mom, Dad {
	void f() override {
		std::cout << 3;
	}
};

int main() {
	 
}
```


Скомпилируется ли? `f` в `Son` должна считаться виртуальной или нет? 
Ошибки не будет, `override` не влияет на виртуальность.

А вот так:
```cpp
struct Mom {
	virtual void f() {
		std::cout << 1;
	}
};

struct Dad {
	virtual void f() {
		std::cout << 2;
	}
};

struct Son : Mom, Dad {
	void f() override {
		std::cout << 3;
	}
};

int main() {
	 Son s;
	 Mom& m = s;
	 Dad& d = s;
	 m.f();  // 3
	 d.f();  // 3
}
```

---

Еще про применение `final`:
```cpp
struct Son final: Mom, Dad {};
```
Теперь от `Son` никому нельзя наследоваться.

## Абстрактные классы
`Abstract classes and pure virtual functions`

_Абстрактный класс_ - это такой класс, у которого есть хотя бы один `pure virtual` метод.

```cpp
struct Shape {
	virtual double area() const = 0;
	virtual ~Shape() = default;  // виртуальный деструктор
};
```

Метод называется `pure virtual`, если он `virtual` и про него написано `= 0`.

Нельзя создать объект абстрактного типа. 

```cpp
struct Shape {
	virtual double area() const = 0;
	virtual ~Shape() = default;  
};

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
	std::vector<Shape*> v;
	v.push_back(new Square(1.0));
	v.push_back(new Circle(1.0));
	
	for (Shape* s: v) {
		std::cout << s->area() << std::endl;
	}
}
```

_Полиморфизм_ - такое средство, когда вы можете, обращаясь к объекту, используя одно и то же имя, название и действие получать разное в зависимости от того, что там было на самом деле.

_Статический полиморфизм_ - это, по сути, перегрузка функций.
_Динамический полиморфизм_ - в примере выше, когда какая именно операция  вызовется определяется в `run time`, а не в `compile time`.
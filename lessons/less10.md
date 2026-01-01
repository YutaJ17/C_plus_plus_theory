
```cpp
#include <iostream>

int main() {
	const int c = 2;

	int x = 5;
	const int y = x;
	int z = y;
}
```

`const` - модификатор типа.
Это другой тип, у которого часть операций исходного типа отсутствует.

```cpp
int main() {
	int x = 5;
	int y = 7;
	int* p = &x;
	
	const int* pc = p;  // константным является только int (указатель на константу)
	*pc = 1;  // CE
	pc = &y;
	
	int* const cp = p;  // константным является только указатель (константный указатель)
	cp = &y;  // CE
	*cp = 1;
	
	const int* const p1 = p;
}
```

```cpp
int main() {
	int x = 5;
	int y = 7;
	int* p = &x;
	
	const int* pc = p;  // int* -> const int* 
	int* p2 = pc;  // CE (обратно нельзя)
}
```

Нет гарантий, что `int` никогда не поменяется. Это только означает, что через разыменовывание нельзя менять значение.
```cpp
int main() {
	int x = 5;
	const int* p = &x;
	
	++x;
	std::cout << *p; // 6
}
```

То есть результатом разыменования `const int*` будет `const int&`, а не `int&`.

`const int*` - право на чтение int
`int*` - право на запись и на чтение int

---

Константные ссылки
```cpp
int main() {
	int x;
	const int& r = x;  // r - это то же самое, что и x, но с меньшими правами
	int& const r1 = x;  // CE
	int& r3 = r;  // CE
	++x;
	std::cout << r; // 6
}
```

`int& const` будет `CE`, так как лишена смысла, ссылка и так является константой, ее нельзя переставить, как указатель.

```cpp
int main() {
	const int x = 5;
	int& y = x;  // CE
	
	const int& r = x;
}
```

Нельзя оставить константу без инициализации:
```cpp
const int c;  // CE
```

Как принять аргумент, по значению, по ссылке или по константной ссылке?
```cpp
using std::string;
string substr(const string& str, size_t from, size_t to);
```

Если нужно из функции менять передаваемый объект, то его нужно принимать по обычной ссылке.
Если нет надобности его менять, то лучше принять по константной ссылке.

```cpp
using std::string;

void f(const string&);  // void f(string&) - ошибка

int main() {
	f("akfsd");  // вызываем f от rvalue
}
```

Обычные ссылки инициализовать через `rvalue` нельзя, а константные можно.

```cpp
void f(const string& s) {
	string s2 = s;
}

int main() {
	{
		// lifeTime expansion 
		const string& s = "abc";
		// ...
		f(s);
	}
}
```

```cpp
void f(const string&);
void f(string&);
void f(string);
```

Примитивные типы нет смысла принимать по константной ссылке.

Если разрешить продлевать жизнь объектам по неконстантной ссылке, то можно, ошибившись в типе, думать, что из функции поменяли объект, а на самом деле поменялась локальная копия. Поэтому это запрещено. 
```cpp
void g(size_t& y) {
	++y;
}

int main() {
	int x = 0;
	g(x);  // CE
	std::cout << x;
}

```

```cpp
using std::string;

void f(string&); // exact type matching
void f(const string&);  // корректная перегрузка

int main() {
	string s = "abc";
	const string& rs = s;
	f(rs);
}
```

`dangling reference`
```cpp
const int& g(int x) {
	return x++;
}

int main() {
	g(5);
}
```

`lifeTime expansion` происходит только при объявлении локальных переменных, но при возврате ссылок из функций. 

---
Так можно?
```cpp
int main() {
	int* p = new int(0);
	int** pp = &p;
	const int** cpp = pp;
}

```
Нет, нельзя неявно добавлять const глубже, чем на 1 уровень под указатель.
Почему нельзя? Если это разрешить:
```cpp
#include <iostream>

int main() {
	const char x = 'a';
	char* p = nullptr;
	const char** q = &p; // q now points to p; this is (fortunately!) is err
	*q = &x; // p now points to x
	*p = 'b'; // ouch: modified a const char
	std::cout << x;

}
```
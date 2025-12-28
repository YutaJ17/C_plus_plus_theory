# CE

```cpp
std::cout << x +;
```
`error`: expected primary expression before `;` token

```cpp
int main() {
	"abc" +5.0f;
}
```
`error`: invalid operands of types

```
++x++;
```
`error`: `lvalue` required as increment operand

# RE

Программа скомпилировалась, то есть из нее получилось сделать исполняемый машинный код, но он падает во время выполнения.

`Segmentation fault` возникает при попытке обращения к памяти, к которой нет права обращаться.

```cpp
#include <vector>
int main() {
	std::vector<int> v(10);
	v[50'000] = 1;  // seg fault: core dumped
}
```

`Floating point exception` - деление целого числа на 0

```cpp
int x;
std::cin >> x;
std::cout << 5 / x;
```

```cpp
float x;
std::cin >> x;
std::cout << 5 / x;  // inf  (ошибки не будет) 
```

`Aborted` - программа аварийно завершается операционной системой

```cpp
#include <vector>
int main() {
	std::vector<int> v(10);
	v.at(10) = 1;
} 
```

# UB

`Underfined behavior` - неопределенное поведение

```cpp
#include <iostream>
#include <vector>

int main() {
	std::vector<int> v(10);
	v[10] = 1;
	
	int x;
	std::cout << x;
	
	x++ + ++x;

}
```

Иногда `UB` приводит к `RE`, иногда к тому, что программа выводит рандомные числа. Компилятор пересдает давать гарантии относильно выполняемого кода.

```cpp
#include <iostream>

int main() {
	for (int j = 0; j < 300; ++j) {
		std::cout << j << ' ' << j * 12345678 << std::endl;  // overflow
	}
}
```

Переполнение `int`, по стандарту - `UB`. 
Что забавно, переполнение `unsigned int` не является `UB`. 

Компилятор, следуя стандарту языка, следует предположению, что `UB` в коде нет.  

Флагом `-02` можно включить оптимизацию при компиляции.

Стандарт C++ формально определяет наблюдаемое поведение каждой программы, кроме тех, которые фейлятся по одной из следующих причин:
1.  `ill-formed` - программа имеет `syntax errors` или 
диагностируемые (на этапе компиляции) `semantic errors`
2.  `ill-formed, no diagnostic required` -  программа имеет `syntax errors`, но компилятор не факт что способен их распознать
3. `implementation-defined behavior` - в зависимости от компилятора разное поведение 
4. `unspecified behavior` - программа не упадет, но нет уверенности, что именно произойдет
5. `undefined behavior`

# The as-if rule

`UB` позволяет делать агрессивные оптимизации. Компилятор может оптимизировать код в предположении, что в нем нет `UB`, засчет этого он многократно выигрывает по скорости выполнения у корректных программ.

Компилятор может делать что угодно с кодом, лишь бы наблюдаемое поведение совпадало с ожидаемым. 

# warning

Это замечание, наблюдение компилятора относительно того, что он видит в коде и подозревает, что это может быть проблемой.
`warning` - это не нарушение стандарта
Компиляция с флагом `-Wall` - будут показываться все `warnings`.
Другие флаги: `-Wextra`, `-Wpedantic`, `-Werror`, `-Wno-error=unused-value`, 
`-Wno-unused-value`

```cpp
#inlcude <iostream>
#include <vector>

int main() {
	int x;
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wunused-value"
	std::cout << x;
#pragma GCC diagnostic pop
}
```
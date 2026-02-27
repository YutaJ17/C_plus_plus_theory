## std::deque internals

```cpp
vector<int> v(10);
int* p = &v[s];
v.push_back(1);
cout << *p;  // UB
```
`Pointer invalidation`: после `push_back` произошла реаллокация и теперь p указывает на память, которая нам не принадлежит.


```cpp
vector<int> v(10);
int& p = v[s];
v.push_back(1);
cout << *p; 
```
`UB` или все ok? Ровная та же ситуация, `UB`. `Reference invalidation`.

**Deck** не инвалидирует указатели и ссылки, в отличие от вектора.

Заведем массив, состоящий из `nullptr`, при надобности указывающий на массивы (buckets) с константным размером. `T** arr`. При реаллокации реаллоцируется внешний массив, а не внутренний. Поддерживается `operator[]` за `O(1)`. Индексация всегда с 0.

`std::stack`, `std::queue`, `std::priority_queue` - адаптеры над `std::deque` (по умолчанию)

```cpp
template <typename T, typename std::deque<T>>
class stack {};
```

## Итераторы и их категории

Не для всех контейнеров можно в качестве индексов использовать целочисленные индексы.
Итератор - это такой тип, который позволяет (неформально говоря) делать обход последовательности. Это некоторое обобщение указателя, которое позволяет себе разыменовывать и инкрементировать, и таким образом делать обход последовательности.


**Input Iterator**
- Самый минималистичный итератор
- Только однократное чтение

**Forward Iterator**
- Частный случай Input Iterator
- Гарантирует одинаковый результат при многократном проходе
- Пример (не Bidirectional): `forward_list`, `unordered_map`

**Bidirectional Iterator**
- Частный случай Forward Iterator
- Умеет: `++it`, `--it`
- Пример (не Random Access): `list`, `set`, `map`

**Random Access Iterator**
- Частный случай Bidirectional Iterator
- Дополнительные возможности:
  - Арифметика: `it += n`, `it -= n`
  - Смещение: `it + n`, `it - n`
  - Доступ по индексу: `it[n]`
  - Сравнение: `<`, `>`, `<=`, `>=`
  - Разность: `it1 - it2`
- Пример: `deque`

**Contiguous Iterator** (C++17)
- Частный случай Random Access Iterator
- Гарантирует последовательное расположение элементов в памяти
- Пример: `vector`, `array`

## Схема отношений
Input → Forward → Bidirectional → Random Access → Contiguous (C++17)

Каждый контейнер STL определяет свой тип итератора в зависимости от своей структуры.


Как по итератору понять, какой у него вид? Как из-под итератора что-то достать?

Алгоритм `find most often number`.

```cpp
#include <iostream>
#include <vector>

template <typename InputInterator>
void find(InputIterator begin, InputIterator end) {
    auto x = *begin;
}

int main() {
    std::vector<bool> vb(10);
    f(*vb.begin());    // BitReference, CE
}
```

Есть набор метафункций, используя которые можно узнать много чего про итераторы. 
Называется `iterator_traits`.



```cpp
#include <iostream>
#include <vector>

template <typename InputInterator>
void find(InputIterator begin, InputIterator end) {
    typename std::iterator_traits<InputIterator>::value_type x = *begin;
}

int main() {
    std::vector<bool> vb(10);
    f(*vb.begin()); 
}
```
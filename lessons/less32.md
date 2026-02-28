## std::list internals

В этих контейнерах нет индексации, нужно абстрагироваться от этого и мыслить итераторами.

Guaranteed `O(1)`:
- `insert(iterator it, const T& value)`
- `erase(iterator it)`
- `push_back/pop_back`
- `push_front/pop_front`
- `size()`

iterator type: `bidirectional iterator`


```cpp
template <typename T>
class list {
    struct Node {
        T value;
        Node* next;
        Node* prev;
    };

    Node* beg;  // begin
    size_t sz;
};
```

В `list` есть фиктивная нода, связывающая `tail` и `end`.

А как в `fake Node` хранить `T value`? list и так медленный тем, что каждый раз происходит `new`. Давайте сделаем так:


```cpp
template <typename T>
class list {
    struct BaseNode {
        BaseNode* next;
        BaseNode* prev;
    };

    struct Node : BaseNode {
        T value;
    };

    BaseNode fakeNode;  
    size_t sz;

public:
    list(): fakeNode {&fakeNode, &fakeNode}, sz(0) {}
};
```
Хочется, чтобы пустой `list` ничего не аллоцировал, при этом `end` и `begin` были равны и к ним можно было обращаться.
Для этого будем хранить `fakeNode` на стеке.

- `sort()` 
- `merge(...)`
- `splice(it, form, to)` - вырезать кусок из одного листа и вставить в другой лист


## std::forward_list

Можно вставлять только в начало, и удалять только из начала. Нет методов `size()`, 
`insert()`.

`LRUCache`


## std::map internals

Упорядоченный ассоциативный массив, ключ-значение.

Guaranteed `O(logn)`:
- `pair<iterator, bool> insert(const pair<key, value>&)`
- `size_t erase(const key&)`
- `value& at(const key&)`
- `const value& at(const key&)`
- `value& operator[](const key&)`
- `iterator erase(iterator)`
- `iterator find(key)`
- `size_t count(const key&)`
 
У метода `operator[]` нет константной версии, потому что `[]` не кидают `exception`, а создают новый ключ с значением по умолчанию, то есть он вызывает неконстантные операции.

```cpp
if (auto it = m.find(1); it != m.end()) {
    it->second = 2;
}
```

```cpp
template <typename T>
class map {
    struct Node {
        pair<key, value> kv;
        Node* left;
        Node* right;
        Node* parent;
        bool red;
    };
};
```

`begin` - самый левый ребенок.
`end` - fakeNode, будет родителем корня дерева.


```cpp
template <typename T>
class map {
    struct BaseNode {
        Node* left;
        Node* right;
        Node* parent;
    };
    struct Node : BaseNode {
        pair<const key, value> kv;
        bool red;
    };
};
```

- `lower_bound`
- `upper_bound`

Если бы не `const key`, то было бы можно использовать алгоритмы, по типу `sort` над `map`.


```cpp
template <typename Key, typename Value, typename Compare = std::less<key>>
class map {
    struct BaseNode {
        Node* left;
        Node* right;
        Node* parent;
    };
    struct Node : BaseNode {
        pair<const key, value> kv;
        bool red;
    };

    BaseNode* fakeNode;
    BaseNode* begin;
    size_t comp;
};
```

Что если компаратор кинет исключение? Значит нужно вернуть все как было и выйти из `map`.
Но в красно-черном дереве не нужно делать повороты после сравнений, это значит, что если кинуто исключение, то дерево еще не повернуто (не испорчено).
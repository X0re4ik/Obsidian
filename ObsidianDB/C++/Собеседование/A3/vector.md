
## Метериал
* [# Динамический массив | Структуры данных и алгоритмы | Изучение алгоритмов](https://www.youtube.com/watch?v=hAfX4IA8LVo&list=PL_5NbJ27RRd3qCREucPf6Kl5p_rHsUe6S&index=7)
* [16.1 — Introduction to containers and arrays](https://www.learncpp.com/cpp-tutorial/introduction-to-containers-and-arrays/)

## Основы работы `std::vector`

`std::vector` - базовый контейнер, который входит в стандартную библиотеку `std`. `std::vector` хранит данные в одном непрерывном участке памяти. Память выделяется через **аллокатор**.

Типичная реализация `std::vector` содержит указатель на начало выделенной памяти, позицию после создания последнего элемента, позицию после конца выделенного элемента (`libstdc++`), упрощенно их можно представить так (Сгенерировано через Perplexity):

```С++
template <typename T, typename Allocator = std::allocator<T>>
class VectorLike {
private:
    T* begin_;           // начало выделенного буфера
    T* end_;             // позиция после последнего созданного элемента
    T* end_of_storage_;  // позиция после конца выделенного буфера
    [[no_unique_address]] Allocator alloc_;
};
```

Модель представленная выше позволяет с легкостью разобраться в текущем состоянии вектора. Например:

```text
size() = end_ - begin_
capacity() = end_of_storage_ - begin_

0 <= size() <= capacity()
```

Схема расположения указателей:

```text
begin_                            end_of_storage_
  |                                      |
  v                                      v
+------+------+------+------+------+
|  10  |  20  | raw  | raw  | raw  |
+------+------+------+------+------+
                ^
                |
               end_
```

Суммарно пустой вектор (64-битной плаформа) занимает 24 байта **памяти на стеке** (`begin_` + `end_` + `end_of_storage_`). Аллокатор - логически часть `std::vector`; физически может занимать 0 байт для пустого аллокатора благодаря оптимизациям компилятора.

```C++
int main() {
  std::vector<int> x_;
  std::cout << sizeof(x_) << " bytes" << std::endl; // 24 bytes
}
```

`std::vector` - **динмаический массив**, значит он способен к расширению за счёт операций `push_back` и `emplace_back`.

## Устройство в памяти: `size`, `capacity`, contiguous storage

`std::vector` хранит свои элементы в непрерывном участке памяти - так же как обычный массив `T[]`. Для `std::vector`, где `T != bool` гарантируется `contiguous storage`:

```C++
&v[i] == &v[0] + i;
```

* `size()` - отвечает на вопрос: "Сколько всего элементов находится в массиве?"
* `capacity()` - отвечает на вопрос: "Сколько всего элементов выделено под массив?"

Визуально можно воспринимать разницу между `size` и `capacity` так:

```text
size     = 0
capacity = 5

+-------+-------+-------+-------+-------+
|  raw  |  raw  |  raw  |  raw  |  raw  |
+-------+-------+-------+-------+-------+
```

Если пользователь запрашивает вставку 6-ого элмента при `capacity = 5`, то массив будет вынужден выполнить расширения `capacity`, чаще всего (в `libstdc++` и `libc++`) путём `capacity = capacity * 2`. Поэтому размер `capacity` "прыгает": `1 -> 2 -> 4 -> 8 -> 16 -> 32 -> ...` 

## Создание векторов и доступ к элементам

Рассмотрим создание объекта `std::vector` на типовых кейсах:

1) Создание массива с резервированием под 10 элементов:

```C++
std::vector<int> v;
v.reserve(10);
```

2) Создание массива из 5 одинаковых элементов:

```C++
std::vector<int> v(5, 42);
// v == {42, 42, 42, 42, 42}
```

3) Создание массива из 5 значений **по умолчанию**:

```C++
struct User {
    User() {
        std::cout << "User created\n";
    }
};

std::vector<User> users(5); // Будет создано 5 объектов User
```

4) Создание массива из диапазона указателей:

```C++
int array[] = {10, 20, 30, 40, 50};
std::vector<int> v(array, array + 5);
// v == {10, 20, 30, 40, 50}
```
5) Создание указателя из диапазона итераторов:

```C++
std::vector<int> source{10, 20, 30, 40, 50};

std::vector<int> part(
    source.begin() + 1,
    source.begin() + 4
);

// part == {20, 30, 40}
```

При создании массива из итератора руководствуемся правилом: `[fisrt, last)` - то есть конец итератора не включительно

Доступ к элментам можно получить одним из следующих способов:

1) Оператор `[]`

```C++
std::vector<int> source{10, 20, 30, 40, 50};
source[1]
source[10] // UB
```

2) Метод `.at()`

```C++
std::vector<int> source{10, 20, 30, 40, 50};
source.at(1);
source.at(10); // throw std::out_of_range
```

## Добавление, вставка и удаление элементов

### Добавление

* `push_back` - добавить элемент в конец

**Сложность:** `O(1)` - амортизированная сложность, если места нет, то происходит релокация - `O(N)`
**Перегрузки:**

```C++
push_back(const value_type& __x) // (1)
push_back(value_type&& __x)      // (2)
```

`push_back` способен, как к копированию объекта (пример `(1)`), так и к полному владению (пример `(2)`). Отметим, что перегрузка `(2)` "под капотом" выполняет `emplace_back`:

```C++
void push_back(value_type&& __x)
{
	emplace_back(std::move(__x));
}
```

* `emplace_back` - добавление элемента в конец. В оличии от `push_back` констрирование элмента происходит "на месте"

**Сложность:** `O(1)` - амортизированная сложность, если места нет, то происходит релокация - `O(N)`
**Перегрузки:**

```C++
emplace_back(_Args&&... __args);
```

Вырезка из реализации:

```C++
vector < _Tp, _Alloc > ::
  emplace_back(_Args && ...__args) {
    if (this -> _M_impl._M_finish != this -> _M_impl._M_end_of_storage) {
      _GLIBCXX_ASAN_ANNOTATE_GROW(1);
      // ❗ КОНСТРИРОВАНИЕ ОБЪЕКТА ❗
      _Alloc_traits::construct(this -> _M_impl, this -> _M_impl._M_finish,
        std::forward < _Args > (__args)...);
      ++this -> _M_impl._M_finish;
      // ❗ КОНСТРИРОВАНИЕ ОБЪЕКТА ❗
      _GLIBCXX_ASAN_ANNOTATE_GREW(1);
    } else
      _M_realloc_insert(end(), std::forward < _Args > (__args)...);
    #if __cplusplus > 201402 L
    return back();
    
// `construct` создаёт объект `T` по указанному адресу
```

### Вставка

* `insert`

**Сложность:** `O(N)`
**Перегрузки:**

```C++
// Копирование существующего объекта
iterator insert(const_iterator __position, const value_type& __x);
// Пример
std::vector<int> v{10, 30};
v.insert(v.begin() + 1, 20); // {10, 20, 30}

// Перемещение существующего объекта
iterator insert(const_iterator __position, value_type&& __x);
// Пример
std::vector<std::string> v{"A", "C"};
v.insert(v.begin() + 1, std::string{"B"}); // {"A", "B", "C"}

// Вставка из `initializer_list`
iterator insert(const_iterator __position, initializer_list<value_type> __l);
// Пример
std::vector<int> v{10, 40};
v.insert(v.begin() + 1, {20, 30}); // {10, 20, 30, 40}

// Вставка диапазона итераторов
iterator insert(const_iterator __position, _InputIterator __first, _InputIterator __last);
// Пример
std::vector<int> v{10, 40}, extra{20, 30};
v.insert(v.begin() + 1, extra.begin(), extra.end()); // {10, 20, 30, 40}
```

Для диапазона действует такое же правило, как и для всех итераторов  вцелом: `[first, last)`

### Удаление элементов

* clear - полная очистка массива от объектов

**Сложность:** `O(N)`, так как необходимо у каждого объекта вызвать деструктор
**Перегрузки:**

```C++
void clear() _GLIBCXX_NOEXCEPT
{ _M_erase_at_end(this->_M_impl._M_start); }

// _M_erase_at_end
void _M_erase_at_end(pointer __pos) _GLIBCXX_NOEXCEPT {
  if (size_type __n = this -> _M_impl._M_finish - __pos) {
	// ❗ Деструктор для каждого объекта ❗
    std::_Destroy(__pos, this -> _M_impl._M_finish,
      _M_get_Tp_allocator());
    // ❗ Деструктор для каждого объекта ❗
    this -> _M_impl._M_finish = __pos;
    _GLIBCXX_ASAN_ANNOTATE_SHRINK(__n);
  }
}
```

* pop_back - удаление объекта с конца

**Сложность:** `O(1)` 
**Перегрузки:**

```C++
void pop_back() _GLIBCXX_NOEXCEPT {
  __glibcxx_requires_nonempty();
  --this -> _M_impl._M_finish;
  _Alloc_traits::destroy(this -> _M_impl, this -> _M_impl._M_finish);
  _GLIBCXX_ASAN_ANNOTATE_SHRINK(1);
}
```

* erase - удаление объекта в произвольном месте массива

**Сложность:** `O(N)`
**Перегрузки:**

```C++
// Удаление элмента на позиции
iterator erase(const_iterator __position)
{ return _M_erase(begin() + (__position - cbegin())); }
// Пример
std::vector<int> values{10, 20, 30, 40};
values.erase(values.begin() + 1); // values == {10, 30, 40}

// _M_erase
template < typename _Tp, typename _Alloc >
  _GLIBCXX20_CONSTEXPR
typename vector < _Tp, _Alloc > ::iterator
vector < _Tp, _Alloc > ::
  _M_erase(iterator __position) {
    if (__position + 1 != end())
		// Сдвиг всех элментов справа на одну позицию влево
	    _GLIBCXX_MOVE3(__position + 1, end(), __position);
	    --this -> _M_impl._M_finish;
	    _Alloc_traits::destroy(this -> _M_impl, this -> _M_impl._M_finish);
	    _GLIBCXX_ASAN_ANNOTATE_SHRINK(1);
    return __position;
  }
```

```C++
// Удаление диапазона элментов
iterator erase(const_iterator __first, const_iterator __last) {
  const auto __beg = begin();
  const auto __cbeg = cbegin();
  return _M_erase(__beg + (__first - __cbeg), __beg + (__last - __cbeg));
}
// Пример
std::vector<int> values{10, 20, 30, 40, 50};
values.erase(values.begin() + 1, values.begin() + 4); // values == {10, 50}
```

## `reserve`, `resize` и `shrink_to_fit`


## Итераторы, ссылки и invalidation

Главное правило работы с сылками и итераторами - если `std::vector` сделал релакацию памяти, все итераторы, ссылки и указатели становятся невалидными

Основные виды итераторов:
1) Input - данные движутся только вперед

**Пример (Сгенерирован Perplexity)**
```C++
std::istringstream input{"10 20 30"};

std::istream_iterator<int> it(input);
std::istream_iterator<int> end;

while (it != end) {
    int value = *it;
    ++it;
}
```

2) Output - записывает элемент и двигается толкьо вперед

**Пример (Сгенерирован Perplexity)**
```C++

```

3) `Forward` - читать писать, обходить диапазон многократно


4) `Bidirectional` - поддерживает все возможности `Forward`, но включает также движение назад

**Контейнеры:** `std::list`, `std::map`, `std::map`

**Операции:**
```C++
it++        // ✅ Движение вперед
it--        // ✅ Движение назад
it + 1      // ❌ Ошибка компиляции
```

5) `Random-access` - поддерживает "прыжки" по объекту

**Контейнеры:** `std::vector`, `std::deque`, `std::array`

**Операции:**
```C++
it + n;           // ✅ Прыжок на N
it[n];            // ✅ Прыжок на N
it2 - it          // ✅ Расстояния между итераторами
it2 < it == false // ✅ Сравнение
```

**Пример**
```C++
std::vector<int> input{1, 2, 3, 4, 5};
auto it = std::begin(input);
std::cout << *(it + 3) << '\n'; // 4
```

6) `Contiguous` - `Random-access` с гарантией, что данные лежат друг за другом


## Алгоритмы STL и удаление по условию

## `push_back` vs `emplace_back`, copy и move

## Производительность и сложность операций

## Особые случаи: `vector<bool>`, `unique_ptr`, полиморфизм, `vector<int&>`


## Выбор контейнера, API и практика


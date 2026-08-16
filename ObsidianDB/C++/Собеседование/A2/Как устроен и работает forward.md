# Как устроен и работает `std::forward`

Посмотрим на реализацию `std::forward`.

```cpp
template <typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type& __t) noexcept {
    return static_cast<_Tp&&>(__t);
}
```

В реальной стандартной библиотеке есть также перегрузка для rvalue-аргумента:

```cpp
template <typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type&& __t) noexcept {
    static_assert(!std::is_lvalue_reference<_Tp>::value,
                  "bad forward call");
    return static_cast<_Tp&&>(__t);
}
```

Ниже сначала рассмотрим первую, основную перегрузку.

По сравнению с реализацией `std::move` ([Как устроен и работает move) видны несколько значимых отличий:

- Возвращаемый тип — `_Tp&&`; `remove_reference` в нём отсутствует.
- `remove_reference` переместился в тип входного параметра.
- `std::forward` нельзя вызвать без явного указания шаблонного параметра: нужно писать `std::forward<T>(arg)`, а не `std::forward(arg)`.

Ранее мы говорили, что `remove_reference` удаляет ссылку из типа шаблонного параметра. Поэтому первая перегрузка `std::forward` принимает lvalue-ссылку на базовый тип:

```cpp
int x = 10;

std::forward<int&>(x); // forward(int&)
std::forward<int>(x);  // forward(int&)
```

Важный момент: следующие вызовы не компилируются, потому что `_Tp` не может быть выведен из параметра типа `std::remove_reference<_Tp>::type&`:

```cpp
// std::forward(x);  // ошибка: шаблонный параметр _Tp не выводится
// std::forward(10); // ошибка: шаблонный параметр _Tp не выводится
```

Шаблонный аргумент `_Tp` передаётся явно. Обычно им служит тот же `T`, который был выведен у вызывающей шаблонной функции.

Теперь рассмотрим, что возвращает `std::forward` с учётом правила свёртывания ссылок ([Как устроен и работает move > Правила свёртывания ссылок](Как устроен и работает move#Правила свёртывания ссылок)).

```cpp
int x = 10;

std::forward<int&>(x);
// static_cast<int& &&>(x) -> static_cast<int&>(x)

std::forward<int>(x);
// static_cast<int&&>(x)
```

Таким образом, `std::forward<T>(arg)` сохраняет категорию значения, закодированную в `T`:

- если `T` — lvalue-ссылка, результат будет lvalue;
- если `T` — не ссылочный тип или rvalue-ссылка, результат будет xvalue.

Вывод: в обычном коде вне шаблонов `std::forward` действительно редко нужен. Например, `std::forward<int>(x)` по смыслу близок к `std::move(x)`, а `std::forward<int&>(x)` просто возвращает lvalue.

Но вся его мощь раскрывается в шаблонах с forwarding reference.

```cpp
void process(int& x) {
    std::cout << "lvalue\n";
}

void process(int&& x) {
    std::cout << "rvalue\n";
}

template <typename T>
void wrapper(T&& arg) {
    process(arg);
}

int a = 5;
wrapper(a);  // lvalue
wrapper(10); // lvalue
```

`process` всегда получает lvalue. Проблема в том, что `arg` — именованная переменная. Любое именованное выражение является lvalue, даже если его тип — `int&&`.

При вызове `wrapper(10)` параметр `arg` имеет тип `int&&`, но само выражение `arg` всё равно является lvalue. Поэтому без `std::forward` выбирается перегрузка:

```cpp
process(int&)
```

Нужен механизм, который восстановит исходную категорию значения аргумента и выберет подходящую перегрузку. Именно таким механизмом выступает `std::forward`:

```cpp
void process(int& x) {
    std::cout << "lvalue\n";
}

void process(int&& x) {
    std::cout << "rvalue\n";
}

template <typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg));
}

int a = 5;
wrapper(a);  // lvalue
wrapper(10); // rvalue
```

Разберём, что происходит в каждом вызове.

```cpp
wrapper(a);
```

Здесь `T` выводится как `int&`. Поэтому:

```cpp
std::forward<T>(arg)
-> std::forward<int&>(arg)
-> static_cast<int& &&>(arg)
-> static_cast<int&>(arg)
```

В `process` передаётся lvalue.

```cpp
wrapper(10);
```

Здесь `T` выводится как `int`. Поэтому:

```cpp
std::forward<T>(arg)
-> std::forward<int>(arg)
-> static_cast<int&&>(arg)
```

В `process` передаётся xvalue, и выбирается перегрузка с `int&&`.

Итак, `std::move` всегда приводит выражение к `xvalue`, а `std::forward<T>` условно восстанавливает категорию значения, с которой аргумент пришёл в шаблонную функцию. `std::forward` применяют в основном к параметрам forwarding reference и передают ему тот же шаблонный параметр `T`.
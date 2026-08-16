# Как устроен и работает `std::move`

`std::move` часто воспринимают как операцию, которая «перемещает объект». На самом деле это не так: сама по себе она **ничего не перемещает**. `std::move` лишь приводит выражение к категории значения `xvalue`, то есть сообщает компилятору, что объект можно рассматривать как источник ресурсов для перемещения.

Собственно перемещение происходит позднее — когда результат `std::move` передаётся в конструктор перемещения, оператор перемещающего присваивания или другую перегрузку, принимающую rvalue-ссылку.

Типичная реализация выглядит так:

```cpp
template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

В старой форме записи с `typename` она эквивалентна следующему коду:

```cpp
template <typename T>
constexpr typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}
```

## Почему параметр имеет тип `T&&`

В шаблоне функции параметр вида `T&&`, где `T` выводится из аргумента, называется *forwarding reference* (ранее часто использовали термин *universal reference*).

Благодаря этому `std::move` может принять и lvalue, и rvalue:

```cpp
int x = 10;
std::move(x);   // аргумент — lvalue
std::move(10);  // аргумент — rvalue
```

Однако тип `T` будет выведен по-разному:

| Переданный аргумент      | Выведенный `T` | Тип параметра после свёртывания ссылок |
| ------------------------ | -------------: | -------------------------------------: |
| `x` (lvalue типа `int`)  |         `int&` |                     `int& &&` → `int&` |
| `10` (rvalue типа `int`) |          `int` |                                `int&&` |

Это важно для понимания того, зачем нужен `std::remove_reference`.

## Почему наивная реализация не работает

Рассмотрим упрощённую реализацию:

```cpp
namespace my {

template <typename T>
constexpr T&& move(T&& t) noexcept {
    return static_cast<T&&>(t);
}

} // namespace my
```

Проверим её:

```cpp
std::string big_data_1(1'000, '1');
std::string big_data_2;

big_data_2 = my::move(big_data_1);

std::cout << big_data_1.size() << " - " << big_data_2.size() << '\n';
```

При передаче `big_data_1` параметр является lvalue. Поэтому шаблонный аргумент выводится как `T = std::string&`.

Подставим это в сигнатуру:

```cpp
T&&  // std::string& &&
```

После свёртывания ссылок получается:

```cpp
std::string&
```

Следовательно, выражение

```cpp
static_cast<T&&>(t)
```

превращается в приведение к `std::string&`, то есть остаётся lvalue. Поэтому выбирается копирующий оператор присваивания, а не перемещающий:

```cpp
big_data_2 = /* lvalue */;
```

## Правила свёртывания ссылок

В C++ нельзя создать «ссылку на ссылку» как самостоятельный тип, но такие комбинации возникают при подстановке шаблонных параметров. Затем применяется *reference collapsing*:

```cpp
T&  &   -> T&
T&  &&  -> T&
T&& &   -> T&
T&& &&  -> T&&
```

Удобная мнемоника: если среди ссылок есть lvalue-ссылка (`&`), результатом будет `&`. Только комбинация `&& &&` даёт `&&`.

## Роль `std::remove_reference`

Упрощённо `std::remove_reference` можно представить так:

```cpp
template <typename T>
struct remove_reference {
    using type = T;
};

template <typename T>
struct remove_reference<T&> {
    using type = T;
};

template <typename T>
struct remove_reference<T&&> {
    using type = T;
};
```

Этот type trait удаляет ссылочную квалификацию:

```cpp
std::remove_reference_t<int>    // int
std::remove_reference_t<int&>   // int
std::remove_reference_t<int&&>  // int
```

Поэтому при вызове:

```cpp
int x = 10;
std::move(x);
```

выводится `T = int&`, но затем `std::remove_reference_t<T>` становится `int`. Возвращаемый тип — `int&&`, а выражение `x` приводится к `int&&`:

```cpp
static_cast<int&&>(x)
```

Именно удаление ссылки до добавления `&&` предотвращает свёртывание `int& && -> int&` и гарантирует, что `std::move` возвращает rvalue-ссылку на базовый тип.

## Категории значений

Корректнее говорить не «`std::move` возвращает объект», а «`std::move` возвращает ссылку, а выражение вызова имеет категорию `xvalue`».

```cpp
int x = 10;

std::move(x);    // lvalue -> xvalue
std::move(10);   // prvalue -> xvalue
std::move(x + 1); // prvalue -> xvalue
```

Здесь есть небольшая терминологическая поправка: `10` и `x + 1` — это `prvalue`, а не просто «rvalue». `xvalue` и `prvalue` являются подкатегориями `rvalue`.

## Где происходит реальное перемещение

Например:

```cpp
std::string source(1'000, '1');
std::string destination;

destination = std::move(source);
```

`std::move(source)` создаёт `xvalue`, поэтому перегрузка `std::string::operator=(std::string&&)` становится подходящей и обычно выбирается вместо копирующего присваивания. Уже этот оператор выполняет фактическую передачу ресурсов.

После перемещения объект `source` остаётся валидным, но его состояние не определено более точно: для стандартной библиотеки обычно говорят, что оно *valid but unspecified*. Такой объект можно уничтожить, присвоить ему новое значение или вызвать операции, допустимые для объекта в неизвестном состоянии; не следует опираться на его прежнее содержимое или размер.

Например, нельзя использовать размер исходной строки как проверку того, произошло ли перемещение:

```cpp
std::string source(1'000, '1');
std::string destination = std::move(source);

// source.size() может быть 0, а может иметь другое допустимое значение.
```

## Практический вывод

`std::move` — это не команда «перемести объект», а явное преобразование выражения в `xvalue`. Оно позволяет вызвать перегрузки, предназначенные для перемещения.

Эквивалентная учебная реализация:

```cpp
namespace my {

template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}

} // namespace my
```

Использовать `std::move` стоит только тогда, когда вы действительно готовы отдать объект как источник ресурсов и далее не рассчитываете на его прежнее значение.
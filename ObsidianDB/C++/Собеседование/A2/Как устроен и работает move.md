# Как устроен и работает `std::move`

Рассмотрим, как реализован `std::move`:

```cpp
template <typename _Tp>
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept {
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t);
}
```

Первое, что бросается в глаза, — наличие `std::remove_reference`. Второе: по факту `std::move` ничего не перемещает, а только выполняет приведение объекта к `&&`.

Попробуем реализовать свой `my::move()`, который, как кажется, должен заниматься перемещением объекта. Поскольку принцип работы `std::remove_reference` пока неизвестен, реализуем версию без него:

```cpp
namespace my {

template <typename _Tp>
constexpr _Tp&& move(_Tp&& __t) noexcept {
    return static_cast<_Tp&&>(__t);
}

} // namespace my
```

Проверим работу нашего `my::move` на простом примере:

```cpp
std::string bigData1(1'000, '1');
std::string bigData2;

bigData2 = my::move(bigData1);

std::cout << bigData1.size() << " - " << bigData2.size() << std::endl;
// Возможный вывод: 1'000 - 1'000
```

Как видно, ожидаемого перемещения не произошло: был вызван обычный копирующий оператор присваивания. Почему так?

Дело в механизме свёртывания ссылок (*reference collapsing*) в шаблонах.

Этот механизм можно интуитивно объяснить по аналогии с логическим оператором `AND`:

```text
&  — 0
&& — 1

&  &&  -> &
```

То есть если в результате подстановки типов где-либо появляется `&`, итоговым типом будет `&`.

Например:

```cpp
int x = 10;
my::move(x);
```

При передаче `x` шаблонный параметр выводится как `int&`:

```cpp
x                         // lvalue типа int
_Tp = int&

my::move(_Tp&& __t)       // my::move(int& && __t)
                           // после свёртывания: my::move(int& __t)
```

Далее возвращаемое выражение также остаётся lvalue-ссылкой:

```cpp
static_cast<_Tp&&>(__t)   // static_cast<int& &&>(__t)
                           // static_cast<int&>(__t)
```

То есть объект как был lvalue, так им и остался. Поэтому при присваивании выбирается копирование, а не перемещение.

Кажется, что вся магия находится в `std::remove_reference<_Tp>::type`. Взглянем на этот тривиальный class template:

```cpp
template <typename _Tp>
struct remove_reference {
    typedef _Tp type;
};

template <typename _Tp>
struct remove_reference<_Tp&> {
    typedef _Tp type;
};

template <typename _Tp>
struct remove_reference<_Tp&&> {
    typedef _Tp type;
};
```

Обратите внимание: этот класс имеет три версии, и всё, что он делает, — удаляет ссылочную квалификацию. Условно можно сказать:

```cpp
std::remove_reference<int>::type    // int
std::remove_reference<int&>::type   // int
std::remove_reference<int&&>::type  // int
```

Такое удаление ссылки даёт возможность независимо от исходного типа ссылки приводить выражение к rvalue-ссылке, то есть к `xvalue`.

Например:

```cpp
int x = 10;
std::move(x);
```

Здесь `_Tp` выводится как `int&`, но `std::remove_reference<_Tp>::type` становится `int`:

```cpp
std::move(x)

_Tp = int&
std::remove_reference<_Tp>::type = int

static_cast<int&&>(x)
```

Поэтому `std::move` может принимать как lvalue, так и rvalue, но выражение вызова `std::move(...)` всегда имеет категорию значения `xvalue`.

```cpp
int x = 10;

std::move(x);     // lvalue  -> xvalue
std::move(10);    // prvalue -> xvalue
std::move(x + 1); // prvalue -> xvalue
```

Небольшая терминологическая поправка: `10` и `x + 1` — это `prvalue`, а `xvalue` и `prvalue` являются разновидностями `rvalue`.

Важно: `std::move` не выполняет перемещение самостоятельно. Он только делает возможным выбор перегрузки для перемещения:

```cpp
std::string bigData1(1'000, '1');
std::string bigData2;

bigData2 = std::move(bigData1); // выбирается operator=(std::string&&)
```

Именно `std::string::operator=(std::string&&)`, а не `std::move`, передаёт ресурсы из `bigData1` в `bigData2`.

После перемещения `bigData1` остаётся валидным объектом, но его значение не следует считать прежним или предсказуемым. Поэтому его размер нельзя использовать как строгую проверку того, произошло ли перемещение.
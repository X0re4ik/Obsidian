Посмот

```C++
template <typename _Tp>
constexpr _Tp &&
forward(typename std::remove_reference<_Tp>::type &__t) noexcept {
  return static_cast<_Tp &&>(__t);
}
```

По сравнению с реализацией `std::move` ([[Как устроен и работает move]]) мы видим несколько значимых отличий.
1) Возвращаемый тип стал `&&`, пропал `remove_reference` из `return`
2) `remove_reference` переместился в тип входного параметра

Ранее мы говорили, что `remove_reference` избавляет нас от ссылки в типе шаблона, тогда `std::forward` всегда будет получать на вход ссылку на объект.

```C++
int x = 10;
int std::forward;
std::forward(10); // std::forward(int&&) -> std::forward(int&)

```
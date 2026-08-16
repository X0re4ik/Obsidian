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
std::forward(x); // std::forward(int&) -> std::forward(int&)
std::forward(10); // std::forward(int&&) -> std::forward(int&)
```

Теперь рассмотрим, что возаращает `std::forward` с учётом правила свертывания ссылок ([[Как устроен и работает move#Правила свёртывания ссылок]]])

```C++
int x = 10;
std::forward(x);
// static_cast<int& &&>(x) -> static_cast<int&>(x)
std::forward(10);
// static_cast<int&& &&>(x) -> static_cast<int&&>(x)
```

Вывод: АБСОЛЮТНО НЕНУЖНАЯ ВЕЩЬ. ИЗГАТЬ ЕЕ ИЗ СТАНДАРТА!!!

Действительно, в повседневной жизне (вне шаблонов) это не очень полезная вещь, она работает как `echo`-сервер.
Но, вся мощь раскрывается в тот, момент, когда мы встрчаем нечто подобное

```C++
void process(int &x) { std::cout << "lvalue\n"; }
void process(int &&x) { std::cout << "rvalue\n"; }

template <typename T> void wrapper(T &&arg) {
	process(arg);
}
```


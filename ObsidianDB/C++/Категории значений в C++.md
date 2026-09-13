
Каждое значение в `C++` характеризуется двумя независимыми свойствами: _[тип](https://ru.cppreference.com/cpp/language/type "cpp/language/type")_ и категория значения. Каждое выраженеи может находиться **только в одном** из трёх категорий значений: prvalue, xvalue, lvalue

**Иерархия категорий значений:**
```text
							 expression
							 /         \
						glvalue       rvalue
						/    \        /    \
					lvalue   xvalue prvalue xvalue
```

Полный список `lvalue` значений перечислен [здесь](https://ru.cppreference.com/cpp/language/value_category), перечислим лишь наиболее типичные:
* Любое именованнное выражение. Если объект имеет название значит он `lvalue`, в т.ч. объекты параметра шаблона
* Идексирование `a[n]`
* Разыменование указателя `*ptr`
* Выражение элемента объект `a.m`, кроме случаев, когда `a` является `rvalue`, в т.с. `m` также является `rvalue`
* Строковые литералы `Hello Wolrd`
Свойства `lvalue`:
* Адресс `lvalue` можно получить встроенным оператором `&`
Напри
```C++
User &make_user_ref() {
  User u{"Anton"};
  return u;
}

User make_user() {
  return {"Anton"};
}

int main() {
    std::cout << &(make_user_ref().name); // OK
    std::cout << &(make_user().name); // Compiler Error
}
```
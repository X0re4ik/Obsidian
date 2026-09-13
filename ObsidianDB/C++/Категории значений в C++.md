
Каждое значение в `C++` характеризуется двумя независимыми свойствами: _[тип](https://ru.cppreference.com/cpp/language/type "cpp/language/type")_ и категория значения. Каждое выраженеи может находиться только в одном из трёх категорий значений
```text
							 expression
							 /         \
						glvalue       rvalue
						/    \        /    \
					lvalue   xvalue prvalue xvalue
```

Ка
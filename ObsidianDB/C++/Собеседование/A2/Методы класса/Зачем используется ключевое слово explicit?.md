
## До C++11

История `explicit` началась еще со времён `С++98`, тогда  `explicit`применялся только для конструктора из одного аргумента. Начиная с `C++11`, его применение расширилось до конструкторов с несколькими аргументами и для операторов приведения типа.
Для чего нужен был `explicit` - борьба с неявным преобразованием типа. `explicit` запрещает выполнять копирующую инициализацию (`copy initialization`), требуя использовать только прямую (`direct initialization`).

Рассмотрим классы `Simple` и `SimpleExplicit` в контексте `C++98`:
```C++
class Simple {
public:
  Simple(int a) : a_(a) {}
private:
  int a_;
};

class SimpleExplicit {
public:
  explicit SimpleExplicit(int a) : a_(a) {}
private:
  int a_;
};

template <typename S> void someFunc(const S &s) {}

int main(int, char **) {
  Simple s3 = 11;
  // SimpleExplicit se3 = 11; - COMPILE ERROR
  SimpleExplicit se3 = SimpleExplicit(11);

  someFunc<Simple>(11);
  // someFunc<SimpleExplicit>(11); - COMPILE ERROR
  someFunc<SimpleExplicit>(SimpleExplicit(11));
  return 0;
}

```
Мы видим, что отсутствие `explicit` позволяет выполнять совсем не очевидные преобразования, по типу `Simple s3 = 11`. Добавление ключевого слова `explicit` позволяет требовать от разработчика явного указания типа создаваемого объекта.

## C++11

В `C++11` с появлением `uniform initialization` ключевого слово `explicit` начало распространяться и на конструкторы с несколькими параметрами.

Рассмотрим классы `Simple` и `SimpleExplicit` в контексте `C++11`:
```C++
class Simple {
public:
  Simple(int a, int b) : a_(a), b_(b) {}
private:
  int a_;
  int b_;
};

class SimpleExplicit {
public:
  explicit SimpleExplicit(int a, int b) : a_(a), b_(b) {}
private:
  int a_;
  int b_;
};

template <typename S> void someFunc(const S &s) {}

int main(int, char **) {
  Simple s3 = {11, 12};
  // SimpleExplicit se3 = {11, 12}; - COMPILE ERROR
  SimpleExplicit se3 = SimpleExplicit(11, 12);

  someFunc<Simple>({11, 12});
  // someFunc<SimpleExplicit>({11, 12}); - COMPILE ERROR
  someFunc<SimpleExplicit>(SimpleExplicit(11, 12));

  return 0;
}
```

Ситуация аналогично `C++98`, только в формате `uniform initialization`.

Также `C++11` разрешил использование `explicit` для оператора преобразования типа.
```C++
class Simple {
public:
  Simple() {}
  operator bool() const { return true; }
};

class SimpleExplicit {
public:
  explicit SimpleExplicit() {}
  explicit operator bool() const { return true; }
};

int main(int, char **) {
  Simple s7{};
  bool b7 = s7;

  SimpleExplicit se7{};
  // bool be7 = se7; - COMPILE ERROR
  bool be7 = static_cast<bool>(se7);

  return 0;
}
```

## Contextual conversion

Не смотря на то, что `C++11` требует явного преобразования для `explicit` операторов преобразования в стандарте есть исключение, а именно преобразование в логических констектах (`while(se7)`, `if (se7)`):

**Пример:**
```C++
class SimpleExplicit {
public:
  explicit SimpleExplicit() {}
  explicit operator bool() const { return true; }
};

int main(int, char **) {
  SimpleExplicit se7{};

  if (se7) {
    std::cout << "se7 is true" << std::endl;
  }

  return 0;
}
```




https://stackoverflow.com/questions/60646412/what-is-the-usecase-for-explicit-bool

SFINAE
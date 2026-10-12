
```C++
auto s = std::make_shared<S>();
auto i = std::shared_ptr<int>(s, &s->b);
```
1) Первый аргумент определяет с кем мы разделяем владение
2) Второй аргумент определяет, чем мы на самом деле владеем
alignas


## Чем отличается `std::shared_ptr` от `std::make_shared`

Основное отличие между двумя вариантами создания объекта - в контрольных блоках, которые они содержат.

### Немного про контрольный блок

Зачем вообще понадобился контрольный блок? Контрольный блок предлагает несколько решений:
1) Совместное владение объектом. Несколько shared_ptr должны знать о том, сколько их всего создано. Для этого есть один источник правды - контрольный блок
2) Разделение времени жизни счетчика и самого объекта. Сам объект существует пока на него ссылается хотя бы один `shared_ptr`, а контрольный блок живет пока на него ссылается хотя бы один `weak_ptr`
3) Контрольный блок берет на себя всю ответственность за время жизни объекта
4) Поддержка многопоточности. Так как shared_ptr может быть много, а программа конкурентной, то контрольный блок берет на себя задачу решения проблемы гонки данных.

Рассмотрим из чего состоит контрольный блок:
```C++
struct control_block
{
    virtual void Dispose() = 0;
    virtual void Delete() = 0;
    std::atomic<unsigned long> shareds;
    std::atomic<unsigned long> refs;
};
```

Важное уточнение перед тем, как изучать работу контрольного блока: время жизни управляемого объекта и время жизни самого `control_block` - это две разные вещи

* `shareds` - **отслеживает количество `shared_ptr`**, которые владеют объектом
* `refs` - отслеживает **время жизни самого** управляющего блока
* `Dispose` - уничтожает **управляемый** объект
* `Delete` - уничтожает управляющий блок

Важное уточнение: **`refs` не обязательно равен количеству weak_ptr**. Общая формула подсчёта `refs = количество weak_ptr + (shareds > 0 ? 1 : 0)`.

Примерная реализация `prod-like constrol block`:
```C++
struct control_block
{
    std::atomic<unsigned long> shareds{1};
    std::atomic<unsigned long> refs{1};

    virtual void Dispose() noexcept = 0;
    virtual void Delete() noexcept = 0;

    virtual ~control_block() = default;

    void AddShared() noexcept
    {
        shareds.fetch_add(1, std::memory_order_relaxed);
    }

    void ReleaseShared() noexcept
    {
        if (shareds.fetch_sub(1, std::memory_order_acq_rel) == 1)
        {
            Dispose();
            ReleaseWeak();
        }
    }

    void AddWeak() noexcept
    {
        refs.fetch_add(1, std::memory_order_relaxed);
    }

    void ReleaseWeak() noexcept
    {
        if (refs.fetch_sub(1, std::memory_order_acq_rel) == 1)
        {
            Delete();
        }
    }
};
```

### shared_ptr<...>(...)

Рассмотрим контрольный блок, который создаётся при стандартном вызове конструктора (`std::shared_ptr<int>(3)`):
```C++
template<typename T>
struct separate_control_block : control_block
{
    virtual void Destroy() noexcept override
    {
        delete ptr;
    }
    virtual void Delete() noexcept override
    {
        delete this;
    }
    T* ptr;
};
```

Обратим внимание, что контрольный блок `separate_control_block` содержит информацию об объекте, которым владеет `shared_ptr`. Логика достаточно тривиальная, если `shareds == 0` - уничтожить `ptr`, если `refs` - уничтожить контрольный блок

`shared_ptr` и `control block` будут ссылаться на один и тот же объект выделенный в памяти:
![[Диаграмма без названия.drawio.svg]]

### make_shared<...>()

`std::make_shared<T>` использует совершенно другой контрольный блок - `combined_control_block`:
```C++
template<typename T>
struct combined_control_block : control_block
{
	
	template <typename... Args> combined_control_block(Args &&...args) {
		::new (static_cast<void *>(buffer)) T(std::forward<Args>(args)...);
	}
	
    virtual void Destroy() noexcept override
    {
        ptr()->~T();
    }
    virtual void Delete() noexcept override
    {
        delete this;
    }
    T* ptr() { return reinterpret_cast<T*>(buffer); }

    // This buffer holds a "T"
    [[alignas(T)]] char buffer[sizeof(T)];
};
```

Он намного сложнее, постараемся, его разобрать. Выделим сразу странные и непонятные части:
1) Зачем отдельный метод `ptr()`?
2) Почему метод, который ответственен за уничтожение объекта вызывает только деструктор, **без** освобождения объекта `delete ptr`?
3) Зачем выделять buffer размера T?
4) Что такое `[[alignas(T)]]` и почему нельзя написать `char buffer[sizeof(T)];`?



#### Зачем отдельный метод `ptr()`?
Пожалуй, самый простой ответ. Так как мы работаем с buffer-ом, в который скрывает наш объект, нам необходимо casting от одного типа к другому, для этого мы используем `reinterpret_cast<T*>`

#### Почему метод, который ответственен за уничтожение объекта вызывает только деструктор, **без** освобождения объекта `delete ptr`?

Ответ на этот вопрос кроется в том, что выделение самой памяти просходит через `placement new`, а удаление объекта через `delete` вызывает деструктор + освобождение памяти. Блок памяти `buffer` принадлежит именно  `combined_control_block`, именно он отвественнен за освобождение памяти, а освобождение произоёдет при вызове `delete this`.

#### Зачем выделять buffer размера T?

Это и есть главная фишка `combined_control_block`, если `separate_control_block` требовал двойного выделения памяти:
```C++
auto x = new T()
new ControlBlock<T>(x)
```
То, `combined_control_block` хранит объект внутри себя, соответсвенно, память выделяется только под контрольный блок:
```C++
template<typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args)
{
    auto* cb = new combined_control_block<T>(
        std::forward<Args>(args)...
    );

    return shared_ptr<T>(
        make_shared_t{}, // TAG: Приватный конструктор
        cb->ptr(),       // Указатель на T
        cb               // Указатель на control block
    );
}
```


Разобрав две версии контрольного блока можно сделать вывод, что:
1) Контрольный блок по-разному создаётся для `std::shared_ptr` и `std::make_shared`
2) Вариант с `std::make_shared` предпочтительней из-за меньших выделений памяти
3) В основе контрольного блока `combined_control_block` лежит механизм `placement new`


### Где реально полезен weak_ptr

Ка

### Разбор кейсов

#### Вопрос №1
```C++
#include <iostream>
#include <memory>

struct S {
    int a = 42;
    int b = 99;

    ~S() {
        std::cout << "~S\n";
    }
};

int main() {
    auto s = std::make_shared<S>();
    auto i = std::shared_ptr<int>(s, &s->b);
    std::weak_ptr<int> w = i;
// (1)
    s.reset();
// (2)
    auto j = w.lock();
// (3)
    i.reset();
// (4)
    // Точка A

    j.reset();

    // Точка B
}
```
**Вопрос:**
* В точке A чему равны `j.use_count()` и `w.use_count()`? Какое значение даст `*j`, и почему это обращение корректно или некорректно?
В точке  `(1)` у нас `shareds = 2` и `refs = 2`. В точке `(2)` `shareds = 1` и `refs = 2`. В точке `(3)` у нас `shareds = 2` и `refs = 2`. В точке `(4)` `shareds = 1` и `refs = 2`. Соотвественно , в точке `A: shareds = 1; refs = 2`, а в точке `B: shareds = 0; refs = 1`. `use_count()` возвращает количество сильных владельцев в соответствующей группе владения
Рассмотрим точку `A`: в этой точке контрольный блок `j` и `w` активен и `j.use_count() == w.use_count() == 1`. `*j == 99`, так как в точке `A` все еще количество обших ссылок не ноль, соответсвенно `*j` будет ссылаться на объект `S`.
* Когда именно выведется `~S`? Почему тип `shared_ptr<int>` у переменной `j` не мешает уничтожению всего объекта `S`?
Сильный счётчик становится равен нулю внутри `j.reset()`. В этот момент вызывается `~S`. К моменту достижения точки `B` объект `S` уже уничтожен, но управляющий блок остаётся жив благодаря w. "не мешает уничтожению" - не совсем корректная формулировка, так как он мешает уничтожению объекта, так как создание `j` вызвал инкемент `shareds`, что в свою очередь отсрочили вызов деструктора `~S`, но так как `j.reset` уменьшает счётчик ссылок до 0, то действительно в данном лучи не мешает .
* В точке B чему равны `w.expired()` и результат `bool(w.lock())`? Существует ли ещё управляющий блок — и если да, что удерживает его?
`expired()` возвращает `true`, если сильный счётчик равен нулю, и `false`, если сильные владельцы ещё существуют. То есть проверка эквивалентна `use_count() == 0`. `expired` смотрит отличен ли от нуля счётчик `shareds`, если нет, то возвращает `false`. В нашем случае в точке `B` счётчик будет равен нули и `expired == true`. `w.lock()` всегда возвращает `shared_ptr`, но в данном случаи он будет с нулевым указателем, что эквивалентно `false`.

#### Вопрос №2
```C++
#include <iostream>
#include <memory>

struct S {
    int value = 42;

    ~S() {
        std::cout << "~S\n";
    }
};

int main() {
    auto s = std::make_shared<S>();
    std::shared_ptr<int> p(s, nullptr);
    std::weak_ptr<int> w = p;

    s.reset();

    // Точка A

    auto q = w.lock();

    // Точка B

    p.reset();
    q.reset();

    // Точка C
}
```
**Вопрос:**
В точке A чему равны `bool(p)`, `p.use_count()` и `w.expired()`? Жив ли объект `S`?
**Ответ:**
`bool(p) == false`; `p.use_count() == 1`; `w.expired() == false`. Объект `S` еще жив, несмотря на то, что мы явно вызвали `s.reset()`. Дело в том, что в точке `A` `shareds == 1`, а, значит, метод `Destroy` контрольного блока не был вызван.
**Вопрос:**
В точке B чему равны `bool(q)`, `q.use_count()` и `w.use_count()`? Приобрёл ли `q` сильное владение при вызове `w.lock()`?
**Ответ:**
`bool(q) == false`; `q.use_count() == 2`; `w.use_count() == 2`. Да, приобрел, все потому что `.lock()` увеличил количество общих ссылок с 1 до 2-ух.
**Вопрос:**
Когда именно выведется `~S`: при `s.reset()`, `p.reset()`, `q.reset()` или выходе из `main`? Что остаётся существовать в точке C и чему равно `w.expired()`?
**Ответ:**
`~S` выведется тогда, когда общее число общих ссылок станет равным 0, а это произойдет в точке `C` (сразу после `q.reset()`). В точке `w.expired() == true`, потому что `shareds == 0` и будет вызван метод `Destroy` контрольного блока.

#### Вопрос №3
```C++
#include <iostream>
#include <memory>

int main() {
    int value = 42;

    std::shared_ptr<int> empty;
    std::shared_ptr<int> p(empty, &value);
    std::weak_ptr<int> w = p;

    // Точка A

    auto q = w.lock();

    // Точка B

    p.reset();

    // Точка C

    std::cout << value << '\n';
}
```
**Вопрос:**
В точке A чему равны `bool(p)`, `p.use_count()` и `*p`? Корректно ли разыменование?
**Вопрос:**
В точке B чему равны `w.expired()`, `bool(q)` и `q.use_count()`? Почему `lock()` даёт такой результат, хотя `p.get()` содержит адрес живого `value`?
**Вопрос:**
Уничтожит ли `p.reset()` переменную `value`? Существует ли здесь управляющий блок и что определяет время жизни `value`?


#### Вопрос №4
```C++
#include <iostream>
#include <memory>

struct S {
    int value = 42;

    ~S() {
        std::cout << "~S\n";
    }
};

struct NoDelete {
    void operator()(S*) const noexcept {
        std::cout << "NoDelete\n";
    }
};

int main() {
    auto a = std::make_shared<S>();
    std::shared_ptr<S> b(a.get(), NoDelete{});

    std::weak_ptr<S> wa = a;
    std::weak_ptr<S> wb = b;

    // Точка A

    a.reset();

    // Точка B

    auto c = wb.lock();

    // Точка C

    c.reset();
    b.reset();

    // Точка D
}
```
**Вопрос:**

В точке A чему равны `a.use_count()` и `b.use_count()`? Сколько управляющих блоков создано? Означает ли `a.get() == b.get()`, что они разделяют владение?

**Ответ:**

В точке A:
- `a.use_count() == 1`
- `a.use_count() == 1`
Всего будет создано **2-а управляющих блока**. `a.get() == b.get()`, потому что оба умных указателя разделяют один и тот же указатель. Но храниться он будет внутри контрольного блока, созданного при инициализации объекта `a` (при помощи механизма `placement new`)

**Вопрос:**
Что выведется при `a.reset()`? В точке B чему равны `wa.expired()` и `wb.expired()`? Можно ли корректно прочитать `b->value`?

`a.reset()` вызовется 

**Вопрос:**
В точке C чему равны `bool(c)` и `c.use_count()`? Можно ли прочитать `c->value`? Что выведется при последующих `c.reset()` и `b.reset()` — и будет ли повторно вызван `~S`?


```C++
#include <iostream>
#include <memory>

struct Node {
    int id;
    std::shared_ptr<Node> next;

    explicit Node(int id) : id(id) {}

    ~Node() {
        std::cout << "~Node " << id << '\n';
    }
};

int main() {
    std::weak_ptr<Node> observer;

    {
        auto a = std::make_shared<Node>(1);
        auto b = std::make_shared<Node>(2);

        a->next = b;
        b->next = a;

        observer = a;

        // Точка A
    }

    // Точка B

    auto x = observer.lock();
    x->next.reset();

    // Точка C

    x.reset();

    // Точка D
}
```
**Вопрос:**
В точке A чему равны `a.use_count()` и `b.use_count()`? При выходе из внутреннего блока какие-либо `Node` уничтожатся? Чему равно `observer.expired()` в точке B?
**Вопрос:**
После `auto x = observer.lock()` чему равен `x.use_count()`? Что произойдёт при `x->next.reset()`: какой объект уничтожится и чему после этого будет равно `x.use_count()` в точке C?
**Вопрос:**
Что произойдёт при `x.reset()`? Напиши весь вывод программы в правильном порядке. В точке D остались ли живые объекты `Node` и какие управляющие блоки ещё существуют?

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

    auto j = w.lock();
    i.reset();

    // Точка A

    j.reset();

    // Точка B
}
```
**Вопрос:**
* В точке A чему равны `j.use_count()` и `w.use_count()`? Какое значение даст `*j`, и почему это обращение корректно или некорректно?
* Когда именно выведется `~S`? Почему тип `shared_ptr<int>` у переменной `j` не мешает уничтожению всего объекта `S`?
* В точке B чему равны `w.expired()` и результат `bool(w.lock())`? Существует ли ещё управляющий блок — и если да, что удерживает его?
**Ответ:**
- В моме
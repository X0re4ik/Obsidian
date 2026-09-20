
## Метериал
* [# Динамический массив | Структуры данных и алгоритмы | Изучение алгоритмов](https://www.youtube.com/watch?v=hAfX4IA8LVo&list=PL_5NbJ27RRd3qCREucPf6Kl5p_rHsUe6S&index=7)
* [16.1 — Introduction to containers and arrays](https://www.learncpp.com/cpp-tutorial/introduction-to-containers-and-arrays/)

## Основы работы `std::vector`

`std::vector` - базовый контейнер, который входит в стандартную библиотеку `std`. `std::vector` хранит данные в одном непрерывном участке памяти. Память выделяется через **аллокатор**.

Типичная реализация `std::vector` содержит указатель на начало выделенной памяти, позицию после создания последнего элемента, позицию после конца выделенного элемента (`libstdc++`), упрощенно их можно представить так (Сгенерировано через Perplexity):

```С++
template <typename T, typename Allocator = std::allocator<T>>
class VectorLike {
private:
    T* begin_;           // начало выделенного буфера
    T* end_;             // позиция после последнего созданного элемента
    T* end_of_storage_;  // позиция после конца выделенного буфера
    [[no_unique_address]] Allocator alloc_;
};
```

Модель представленная выше позволяет с легкостью разобраться в текущем состоянии вектора. Например:

```text
size() = end_ - begin_
capacity() = end_of_storage_ - begin_

0 <= size() <= capacity()
```

Схема расположения указателей:

```text
begin_                            end_of_storage_
  |                                      |
  v                                      v
+------+------+------+------+------+
|  10  |  20  | raw  | raw  | raw  |
+------+------+------+------+------+
                ^
                |
               end_
```

Суммарно пустой вектор (64-битной плаформа) занимает 24 байта **памяти на стеке** (`begin_` + `end_` + `end_of_storage_`). Аллокатор - логически часть `std::vector`; физически может занимать 0 байт для пустого аллокатора благодаря оптимизациям компилятора.

```C++
int main() {
  std::vector<int> x_;
  std::cout << sizeof(x_) << " bytes" << std::endl; // 24 bytes
}
```

`std::vector` - **динмаический массив**, значит он способен к расширению за счёт операций `push_back` и `emplace_back`.

## Устройство в памяти: `size`, `capacity`, contiguous storage

`std::vector` хранит свои элементы в непрерывном участке памяти - так же как обычный массив `T[]`. Для `std::vector`, где `T != bool` гарантируется 

## Создание векторов и доступ к элементам




## Добавление, вставка и удаление элементов

## Рост вектора и реаллокация

## `reserve`, `resize` и `shrink_to_fit`

## Итераторы, ссылки и invalidation

## Алгоритмы STL и удаление по условию

## `push_back` vs `emplace_back`, copy и move

## Производительность и сложность операций

## Особые случаи: `vector<bool>`, `unique_ptr`, полиморфизм

## Выбор контейнера, API и практика


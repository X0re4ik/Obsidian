
## Метериал
* [# Динамический массив | Структуры данных и алгоритмы | Изучение алгоритмов](https://www.youtube.com/watch?v=hAfX4IA8LVo&list=PL_5NbJ27RRd3qCREucPf6Kl5p_rHsUe6S&index=7)
* [16.1 — Introduction to containers and arrays](https://www.learncpp.com/cpp-tutorial/introduction-to-containers-and-arrays/)

## Основы работы `std::vector`


std::vector хранит данные в одном непрерывном участке памяти. Память выделяется через **аллокатор**.
Вектор содержит обязательный набор полей, упрощенно их можно представить так (Сгенерировано через Perplexity):

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

Суммарно пустой вектор всегда занимает 24 байта памяти (`begin_` + `end_` + `end_of_storage_`). Аллокатор не учитывается, так как 

## Устройство в памяти: `size`, `capacity`, contiguous storage


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


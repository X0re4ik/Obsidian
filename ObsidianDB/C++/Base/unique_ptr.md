# Введение
unique_ptr - умный указатель, который обеспечивает уникальность владения, то есть объект, которым владеет unqiue_ptr **больше никто владеть не может**
# Отличительные особенности
- Данный вид указателей не имеет конструктора копирования
- Данный вид указателя имеет конструктор перемещения, как раз таки для передачи владения


# Передаем std::unique_ptr правильно
1) Мы хотим передать в функцию объект, которым владеет std::unqiue_ptr.
В подобной ситуации нам проще пережать объект по ссылке или указателю, и внутри функции работать с понятной сущностью ссылки/указателя
```C++
// Подход по ссылке
auto ref_foo(A &_ptr) {
	_ptr.x = 10;
	return;
}

std::unique_ptr<A> y = std::make_unique<A>();
ref_foo(*y);

// Подход по указателю
auto point_foo(A *_ptr) {
	_ptr->x = 12;
	return;
}

std::unique_ptr<A> y = std::make_unique<A>();
point_foo(y.get());
```
2) Полная передача владения
Мы хотим полностью отдать std::unqiue_ptr функции.
```C++
std::vector < std::unique_ptr < A >> g_variables;

auto move_foo(std::unique_ptr < A > && _move) {
  g_variables.push_back(std::move(_move));
  return;
}

int main() {
  std::unique_ptr < A > y = std::make_unique < A > ();
  // Полная передача владения A функции move_foo
  move_foo(std::move(y));
  // После данного блока кода считаем, что объект y пустой.
  std::cout << g_variables.size() << std::endl;
  
  // UB
  std::cout << y -> x << std::endl;
  return 0;
}
```
3) Передача по ссылке

## Применение std::unique_ptr
## Pimpl (Pointer to implementation)
-- todo
## Factory Method
```C++
std::unique_ptr<Sensor> createSensor(const std::string& type) {
    if (type == "Lidar") return std::make_unique<LidarSensor>();
    return std::make_unique<CameraSensor>();
}
```
## RAII-обертки для C API
-- todo


# Чем отличается `std::make_unique` от `new`?
1) Первое и основное правило, мы не хотим использовать в своем коде "сырые" указатели
2) Exception safe, в случаи ошибки на стороне конструктора или же ошибки алокации памяти, то make_unqiue гарантирует нам освобождение ресурса, в случаи обычной передачи указателя такой гарантии нет 
```C++
foo(make_unique<T>(), make_unique<U>()); // exception safe

foo(unique_ptr<T>(new T()), unique_ptr<U>(new U())); // unsafe*
```
 Отметим, что никакого **выйгрыша** в производительности, как например в std::make_shared нет

# Дополнительные расходы при использовании std::unique_ptr
Рассмотрим пример стандартного использования std::unique_ptr:
- Размер объекта и указателя идентичны и равны 8 байтам
```C++
class A {
public:
  int x = 6;
};
int main() {
  A *raw_ptr = new A();
  auto u_ptr = std::make_unique<A>();
  std::cout <<
    //Raw Ptr: 8
    "Raw Ptr: " << sizeof(raw_ptr) << "\n" <<
    //Unique Ptr: 8
    "Unique Ptr: " << sizeof(u_ptr) << std::endl;
  return 0;
}
```
- Значительное изменение размера (в 2 раза) призойдет, если мы попробуем использовать custom deleter:
```C++
class A {
public:
  int x = 6;
};

class CustomDeleterA {
public:
  void operator()(A *a) {
    std::cout << "CustomDeleterA" << std::endl;
    delete a;
  }
};

void funcCustomDeleterA(A *a) {
  std::cout << "funcCustomDeleterA" << std::endl;
  delete a;
}

int main() {
  A *raw_ptr = new A();

  std::unique_ptr<A, CustomDeleterA> u_d1_ptr(new A{});
  std::unique_ptr<A, decltype(&funcCustomDeleterA)> u_d2_ptr(
      new A{}, funcCustomDeleterA);

  std::cout <<
      // Raw Ptr (raw_ptr): 8
      "Raw Ptr: " << sizeof(raw_ptr) << "\n"
            <<
      // Unique Ptr (u_d1_ptr): 8
      "Unique Ptr: " << sizeof(u_d1_ptr) << "\n"
            <<
      // Unique Ptr (u_d2_ptr): 16
      "Unique Ptr: " << sizeof(u_d2_ptr) << std::endl;

  return 0;
}
```
Разберем данный кейс более подробно, а именно, почему в случаи `u_d1_ptr` размер составил 8 байт, а в случаи `u_d2_ptr` целых 16. Все дело в объекте std::tuple и механизму EBCO, который "схлопывает" пустой класс.
```C++
tuple<pointer, _Dp> _M_t; // **Empty Base Class Optimization (EBCO)**
```
# Как правильно возвращать std::unique_ptr
```C++
auto create_unique_ptr() {
  std::unique_ptr<A, CustomDeleterA> u_d1_ptr(new A{});
  return u_d1_ptr;
}
```
При возврате объекта использовать дополнительные std::move и прочее не нужно, так как компилятор всегда старается применить RVO, если есть конструктор перемещения (а он есть у std::unique_ptr, то компилятор применить перемещение)
Единственное исключение:
```C++
class A {
public:
  int x = 6;

  std::unique_ptr<int> u_d1_ptr;
};

std::unique_ptr<int> create_unique_ptr() {
  A a = A();
  return std::move(a.u_d1_ptr);
}
```
В этом случаи мы должны забрать владение у локальной переменной A и отдать ее в `return` (Данный пример является антипатерном)

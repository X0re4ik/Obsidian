
## Что такое принцип KISS (Keep It Simple, Stupid)? Приведи пример кода, который его нарушает.

**KISS** - принц, который говорит нам быть "попроще", ООП это все хорошо, но код нужен только для того, чтобы решить бизнес задачу, если ее можно решить быстро и эффективно одним действие, то не нужно плодить много классов и функций.

```C++
class BinaryOperation {
public:
	virtual int calculate(int x, int y) = 0; 
};

class SumOperation : public BinaryOperation {
public:
	virtual int calculate(int x, int y) override {
		return x + y;
	}
};

auto op = SumOperation();
auto res = op.calculate(1, 4);
// OR
res = 1 + 4;
```

## Что такое принцип DRY (Don't Repeat Yourself)? Какие проблемы возникают при дублировании кода?

**DRY** - не нужно повторяться, если есть общий код, то его нужно вынести в общий метод. Такой подход упрощает тестирование, ведь тестирование 5 различных (но очень похожих методов), мы тестируем один с 5 разными кейсами

```C++
// Обработка обычного заказа
double totalOrder = orderPrice + (orderPrice * 0.20);
if (orderPrice > 1000) {
    totalOrder -= 50;
}

// Обработка счета для юридических лиц в другом файле
double totalInvoice = invoicePrice + (invoicePrice * 0.20);
if (invoicePrice > 1000) {
    totalInvoice -= 50;
}

// OR

double calculateTotalPrice(double basePrice, double taxRate = 0.20) {
    double total = basePrice * (1.0 + taxRate);
    if (basePrice > 1000.0) {
        total -= 50.0;
    }
    return total;
}

double totalOrder = calculateTotalPrice(orderPrice);
double totalInvoice = calculateTotalPrice(invoicePrice);
```


## Что такое принцип YAGNI (You Aren't Gonna Need It)? Какие риски несёт добавление функциональности «про запас»?

**YAGNI** - реализовывать только те функции и методы, которые нужны для конкретной бизнес задачи, **хорошо** писать расширяемый код, но не стоит его расширять без потрбности бизнеса в расширении.

Риски кода «про запас»:
* Когнитивная нагрузка - другой разработчик будет вынужден изучать и смотреть, что тут написано и наличие дополнительного метода "про запас" вызовет закономерный вопрос - ЗАЧЕМ?
* Потеря времени - в эпоху ИИ все требуют ускорение бизнес-процессов, лишние функции ее только губят
* Тестирование мертвого кода - в дорогих системах требование к тестированию жесткие и придется писать тесыт на код, который никто не использует

## Могут ли KISS и DRY вступать в противоречие друг с другом? Приведи пример.

- KISS требует: сделай проще, не плоди абстракции
- DRY требует: убери дублирование, вынеси в одно место

```C++
// KISS
void sendToAudit(Producer* producer, )
```

## Что такое принцип KISS (Keep It Simple, Stupid)? Приведи пример кода, который его нарушает.

**KISS** - принцип, призывающий выбирать простое решение, достаточное для задачи. Не следует добавлять лишние классы, уровни абстракций и усложнения, если они не улучшают код и не нужны бизнесу.

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

**DRY** - не дублируй знания и правила в коде. Повторяющуюся логику стоит вынести в одну функцию или компонент, чтобы изменение правила не требовало исправлять несколько мест. Это также уменьшает расхождения и объём тестирования.

```C++
// Обработка обычного заказа
double totalOrder = orderPrice + (orderPrice * 0.20);
if (orderPrice > 1000) {
    totalOrder -= 50;
};

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

**YAGNI** - реализуй только то, что действительно нужно текущей задаче. Расширяемость полезна, но добавление функций «про запас» оправдано только при реальной потребности.

Риски кода «про запас»:
* Увеличивается сложность и когнитивная нагрузка.
* Разработка и тестирование занимают время без подтверждённой пользы.
* Мёртвый код нужно поддерживать, а будущие требования всё равно могут оказаться другими.

## Могут ли KISS и DRY вступать в противоречие друг с другом? Приведи пример.

- KISS требует: сделай проще, не плоди абстракции
- DRY требует: убери дублирование, вынеси в одно место
```C++
// KISS
void sendToAudit(Producer* producer, std::string topic, std::string message) {
	auto msg = "[AUDIT] " + message;
	producer->send_and_wait(topic, message);
}

void sendToServiceX(Producer* producer, std::string topic, std::string message) {
	producer->send_and_wait(topic, message);
}
```
В примере выше есть небольшое дублирование, но два метода явно выражают разные сценарии. Попытка устранить его одной универсальной функцией может ухудшить читаемость и привести к флагу `isAudit`. Поэтому KISS и DRY нужно применять с учётом контекста:
```C++

struct KafkaDataSend {
	Producer* producer;
	std::string topic;
	std::string message;
	bool isAudit;
}

void sendToKafka(KafkaDataSend& kafkaData) {
	auto msg = kafkaData.message;
	if (kafkaData.isAudit) {
		msg = "[AUDIT] " + msg;
	}
	kafkaData.producer->send_and_wait(kafkaData.topic, msg);
}
```

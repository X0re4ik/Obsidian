
## Что такое принцип KISS (Keep It Simple, Stupid)? Приведи пример кода, который его нарушает.

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
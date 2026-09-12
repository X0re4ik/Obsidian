
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

```
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


```
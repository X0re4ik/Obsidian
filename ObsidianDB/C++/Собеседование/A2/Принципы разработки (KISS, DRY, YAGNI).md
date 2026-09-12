
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
op.
// OR

x = 

```
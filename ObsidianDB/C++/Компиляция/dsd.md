#include <iostream>

int func()
{
    static int counter = 5;
    return ++counter;
}

int main()
{
    int result = 0;
    for (int i = 0; i < 10; ++i)
    {
        result = func();
    }

    std::cout << result << '\n';
    return 0;
}

---

#include <iostream>

int main()
{
    int a = 1;
    int& b = a;
    b = 2;
    int* c = &b;
    *c = 3;
    std::cout << a << b << *c << '\n';
// 3 3 3
    return 0;
}

---

#include <iostream>

int main()
{
    char c1 = 'a';
    decltype(c1) c2 = c1;
    ++c2;
    decltype((c1)) c3 = c1;
    ++c3;
    // b b b
    std::cout << c1 << c2 << c3 << '\n';
    return 0;
}

---

#include <iostream>

template <typename T>
void f(T)
{
    static int i = 0;
    std::cout << ++i;
}

int main()
{
    f(1);
    // 1
    f(1.0);
    // 1
    f(1);
    // 2
}

---

templte<typename T>
T sum(const T& a, T b) {
    return a + b;
}

---

struct Test
{
    int i = 123;
    const int& get() const
    {
        return i;
    }
};

int main()
{
    Test t;
    auto q = t.get();
}

---

// TODO
#include <cstdint>
#include <iostream>

#pragma pack(4)

struct A
{
    std::uint64_t Value1 = 10;
    std::uint32_t Value2 = 20;
    std::uint16_t Value3 = 30;
};

int main()
{
    std::cout << sizeof(A) << '\n';
}

---

#include <stdio.h>

#define SQUARE(a) (a) * (a)

int main()
{   
    // (4) * (4) 16
    printf("%d\n", SQUARE(4));
    int x = 3;
    // (++x) * (++x)
    // 4 * 4
    printf("%d\n", SQUARE(++x));
}

---

char (*x)(char*);

---

#include <stdio.h>

int main()
{
    float f = 1.0;

    int i1 = (int)f;
    int i2 = *(int*)&f;
    
    // 1
    printf("%d\n", i1);

    // 0 1100101 000000000
    // 453434
    printf("%d\n", i2);
}

---

size_t my_strlen(const char* str) {
    size_t len = 0;
    while(str[len]) {
        ++len;
    }
    return len;
}

---

#include <iostream>

int main()
{
    unsigned char n = 256; // 0
    for (unsigned char i = 0; i < n; ++i)
    {
        std::cout << "a";
    }

    unsigned char h = 128; // 128
    for (unsigned char i = 0; i < 2 * h; ++i)
    {
        std::cout << "b"; //TODO
    }
}

---

// TODO
#include <iostream>

int main()
{
    int a = 0;
    const int& ra = a;
    const double& dra = a;
    a++;
    std::cout << a;
    std::cout << ra;
    std::cout << dra;
}



































































Материал: 
https://habr.com/ru/companies/otus/articles/801123/
https://www.opennet.ru/base/dev/valgrind_memory.txt.html


## Как Valgrind исполняет мою программу?

Valgrind не просто наблюдает за работающей программой, а запускает ее через собсвтенную среду исполнения. Все инструкции машинного кода, котороые исполняет программ проходят через Valgrind.

**Например:**
```text
Было:
app → CPU → Linux kernel / память / устройства

Стало:
app → Valgrind → CPU → Linux kernel / память / устройства
```




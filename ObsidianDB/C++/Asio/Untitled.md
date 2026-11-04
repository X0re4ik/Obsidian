


boost::asio::io_context - объект, который связывает программу с асинхронными механизмами ОС. Очередь событий + треды, которые ее рашребают

https://www.boost.org/doc/libs/1_53_0/doc/html/boost_asio/examples.html


**Proactor vs Reactor**

| Сигнал      | Номер  | Способ отправки                                  |
| ----------- | ------ | ------------------------------------------------ |
| **SIGHUP**  | **1**  | `kill -1 <PID>` или `kill -HUP <PID>`            |
| **SIGINT**  | **2**  | Нажать `Ctrl+C` в терминале или `kill -2 <PID>`  |
| **SIGQUIT** | **3**  | Нажать `Ctrl+\` в терминале или `kill -3 <PID>`  |
| **SIGTERM** | **15** | `kill <PID>` (по умолчанию) или `kill -15 <PID>` |

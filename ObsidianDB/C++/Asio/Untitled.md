


boost::asio::io_context - объект, который связывает программу с асинхронными механизмами ОС. Очередь событий + треды, которые ее рашребают

https://www.boost.org/doc/libs/1_53_0/doc/html/boost_asio/examples.html


**Proactor vs Reactor**

- `SIGINT` — Ctrl+C в терминале
    
- `SIGTERM` — «аккуратно завершись» от systemd/kill (дефолт для `systemctl stop`)
    
- `SIGHUP` — часто используется для «перезагрузи конфиг»
    
- `SIGQUIT` — тоже завершение, но с дампом ядра по дефолту


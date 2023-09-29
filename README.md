# Домашнее задание к занятию "6.6. Troubleshooting" Соловьев Д.В.

## Задача 1

> Перед выполнением задания ознакомьтесь с документацией по [администрированию MongoDB](https://docs.mongodb.com/manual/administration/).
> 
> Пользователь (разработчик) написал в канал поддержки, что у него уже 3 минуты происходит CRUD операция в MongoDB и её нужно прервать. 
> 
> Вы как инженер поддержки решили произвести данную операцию:
> - напишите список операций, которые вы будете производить для остановки запроса пользователя
> - предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

### напишите список операций, которые вы будете производить для остановки запроса пользователя

1. Найду запрос
   ```js
   db.currentOp({ "active" : true, "secs_running" : { "$gt" : 180 }})
   ```
   Он должен вернуть что-то вроде этого:
   ```js
   {
       "inprog" : [
           {
               //...
               "opid" : 29833,
               "secs_running" : NumberLong(436)
               //...
           }
       ]
   }
   ```
1. [Завершу](https://docs.mongodb.com/manual/tutorial/terminate-running-operations/#killop) принудительно
   ```
   db.killOp(29833)
   ```

### предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

- Использовать метод [`maxTimeMS()`](https://docs.mongodb.com/manual/tutorial/terminate-running-operations/#maxtimems)
- Включить [профайлер](https://docs.mongodb.com/manual/tutorial/manage-the-database-profiler/) чтобы отловить медленные запросы, изучить их с помощью [explain](https://docs.mongodb.com/manual/reference/explain-results/#executionstats) и попробовать оптимизировать: денормализовать данные, добавить/удалить индексы, настроить шардинг и т.д.

## Задача 2

> Перед выполнением задания познакомьтесь с документацией по [Redis latency troobleshooting](https://redis.io/topics/latency).
> 
> Вы запустили инстанс Redis для использования совместно с сервисом, который использует механизм TTL. 
> Причем отношение количества записанных key-value значений к количеству истёкших значений есть величина постоянная и увеличивается пропорционально количеству реплик сервиса. 
> 
> При масштабировании сервиса до N реплик вы увидели, что:
> - сначала рост отношения записанных значений к истекшим
> - Redis блокирует операции записи
> 
> Как вы думаете, в чем может быть проблема?

### Как вы думаете, в чем может быть проблема?

Если я правильно понял [эту статью](https://docs.redis.com/latest/rs/concepts/memory-performance/memory-limit/), то при добавлении очередной реплики, мы превысили лимит памяти `maxmemory`, в таком случае Redis [отвечает ошибкой записи](https://redis.io/topics/faq).

## Задача 3

> Перед выполнением задания познакомьтесь с документацией по [Common Mysql errors](https://dev.mysql.com/doc/refman/8.0/en/common-errors.html).
> 
> Вы подняли базу данных MySQL для использования в гис-системе. При росте количества записей, в таблицах базы, пользователи начали жаловаться на ошибки вида:
> ```python
> InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
> ```
> 
> Как вы думаете, почему это начало происходить и как локализовать проблему?
> 
> Какие пути решения данной проблемы вы можете предложить?

Ошибка

> InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '

### Как вы думаете, почему это начало происходить и как локализовать проблему?

Описание задачи намекает на причину: таблица разрослась, в таком случае запросы могут выполняться долго. В сообщении упоминается `SELECT`, возможно в таблице просто не хватает индексов. 

Чтобы найти такие запросы, можно попробовать включить [slow_query_log](https://dev.mysql.com/_doc_/refman/8.0/en/server-system-variables.html#sysvar_slow_query_log).

В документации MySQL ошибке посвящена [эта](https://dev.mysql.com/doc/refman/8.0/en/error-lost-connection.html) статья, в ней перечислены три возможные причины:
* Слишком объёмные запросы на миллионы строк, и рекомендуют увеличить параметр `net_read_timeout`
* Небольшое значение параметра `connect_timeout`, клиент не успевает установить соединение
* Размер сообщения/запроса превышает размер буфера, заданного в переменной ` max_allowed_packet` [на сервере](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_allowed_packet) или опцией `--max_allowed_packet` [клиента](https://dev.mysql.com/doc/refman/8.0/en/packet-too-large.html) 

Ещё можно предположить, что проблема в неактивности клиента. Это строга из лога SQLAlchemy, на [StackOverflow](https://stackoverflow.com/questions/29755228/sqlalchemy-mysql-lost-connection-to-mysql-server-during-query) предложили уменьшить параметр `pool_recycle` в SQLAlchemy и увеличить `wait_timeout` в [настройках](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_wait_timeout) сервера MySQL.

### Какие пути решения данной проблемы вы можете предложить?

Поправить все перечисленные параметры:
* Увеличить на сервере MySQL `wait_timeout`, `max_allowed_packet`, `net_write_timeout` и `net_read_timeout`
* В SQLAlchemy уменьшить `pool_recycle`, сделать меньше `wait_timeout`

Если ошибка пропадёт, возвращать по одному в исходное состояние, так должно получиться локализовать проблему.

Возможно достаточно будет оптимизировать запросы. Их можно найти включив `slow_query_log` и [настроив](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_long_query_time) параметр `long_query_time` на значение близкое к `net_write_timeout` и `net_read_timeout`. 

Если проблема происходит только на селектах, вероятно изменения таймаутов и оптимизации запросов будет достаточно. Если ещё на инсертах - возможно, наоборот, нужно удалить часть индексов. 

## Задача 4

> Перед выполнением задания ознакомтесь со статьей [Common PostgreSQL errors](https://www.percona.com/blog/2020/06/05/10-common-postgresql-errors/) из блога Percona.
> 
> Вы решили перевести гис-систему из задачи 3 на PostgreSQL, так как прочитали в документации, что эта СУБД работает с большим объемом данных лучше, чем MySQL.
> 
> После запуска пользователи начали жаловаться, что СУБД время от времени становится недоступной. В dmesg вы видите, что:
> 
> `postmaster invoked oom-killer`
> 
> Как вы думаете, что происходит?
> 
> Как бы вы решили данную проблему?

В dmesg вы видите, что:
> 
> `postmaster invoked oom-killer`

### Как вы думаете, что происходит?

Посгтгресу не хватает памяти.

### Как бы вы решили данную проблему?

Percona [предлагают](https://www.percona.com/blog/2019/08/02/out-of-memory-killer-or-savior/) настроить "перевыделение" памяти в `sysctl.conf`:
* `vm.overcommit_memory = 2`
* `vm.overcommit_ratio` без рекомендации, по-умолчанию 60%; если вся БД помещается в ОЗУ, на Хабре [советуют](https://habr.com/ru/company/southbridge/blog/464245/) поставить = 1

Та же можно настроить swap: указать в параметре `vm.swappiness` значение побольше (по-умолчанию = 60), тобы ядро начало использовать своп раньше.

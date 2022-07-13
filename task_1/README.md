# Лекция 2: SQL и реляционные СУБД. Введение в PostgreSQL

## Домашнее задание. Выполнил Балабанов Дмитрий.

### Поставить PostgreSQL

docker-compose.yml
```yaml
version: '3.1'
services:
  db:
    image: postgres:14
    restart: always
    environment:
      POSTGRES_DB: task1
      POSTGRES_HOST_AUTH_METHOD: trust
    ports:
      - "5555:5432"

```
Результат:
```
dmitry.balabanov@MacBook-Pro-X5 task_1 % docker-compose up                      
Creating network "task_1_default" with the default driver
Creating task_1_db_1 ... done
Attaching to task_1_db_1
.....
.....
db_1  | 2022-07-13 12:02:04.690 UTC [1] LOG:  database system is ready to accept connections
```

### Зайти вторым ssh (вторая сессия), запустить везде psql из под пользователя postgres
```
dmitry.balabanov@MacBook-Pro-X5 task_1 % psql -U postgres -h localhost -p 5555 -d task1
psql (14.1, server 14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

task1=# 
```

### Выключить auto commit
```
task1=# \echo :AUTOCOMMIT
on
task1=# \set AUTOCOMMIT OFF
task1=# \echo :AUTOCOMMIT
OFF
```

### Сделать в первой сессии новую таблицу и наполнить ее данными
```
task1=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
task1=*# insert into persons(first_name, second_name) values('ivan', 'ivanov'), ('petr', 'petrov');
INSERT 0 2
task1=*# commit;
COMMIT
task1=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

task1=*# 
```

### Посмотреть текущий уровень изоляции
```
task1=*# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

### Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
```
task1=# begin;
BEGIN
task1=*# 
```

### В первой сессии добавить новую запись
```
task1=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
task1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

task1=*# 
```

### Сделать select * from persons во второй сессии
```
task1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

task1=*# 
```

### Видите ли вы новую запись и если да то почему?
Новой записи не видно, т.к в транзакции, работающей на уровне *read committed*, запрос SELECT 
(без предложения FOR UPDATE/SHARE) видит только те данные, которые были зафиксированы до начала запроса.
(https://postgrespro.ru/docs/postgrespro/14/transaction-iso#XACT-READ-COMMITTED)

### Завершить первую транзакцию
```
task1=*# commit;
COMMIT
task1=# 
```

### Сделать select * from persons во второй сессии
```
task1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

task1=*#
``` 

### Видите ли вы новую запись и если да то почему?
Новая запись видна, т.к запрос во второй сессии выполнен после фиксации транзакции в первой сессии

### Завершите транзакцию во второй сессии
```
task1=*# commit;
COMMIT
task1=# 
```

### Начать новые но уже repeatable read транзации
```
task1=# begin;
BEGIN
task1=*# set transaction isolation level repeatable read;
SET
task1=*# 
```

### В первой сессии добавить новую запись
```
task1=# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
task1=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

task1=# 
```

### Сделать select * from persons во второй сессии
```
task1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

task1=*# 
```

### видите ли вы новую запись и если да то почему?
Новая запись не видна

### завершить первую транзакцию
```
task1=*# commit;
COMMIT
task1=#
```

### сделать select * from persons во второй сессии
```
task1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

task1=*# 
```

### видите ли вы новую запись и если да то почему?
Новая запись не видна

### завершить вторую транзакцию
```
task1=*# commit;
COMMIT
task1=#
```

### сделать select * from persons во второй сессии
```
task1=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

task1=# 
```

### видите ли вы новую запись и если да то почему?
На уровне Repeatable Read не видны изменения, произведенные другими транзакциями до фиксации текущей транзакции.
После коммита транзакции второй консоли изменения первой стали видны. 
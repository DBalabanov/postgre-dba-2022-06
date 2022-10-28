# Логический уровень PostgreSQL

### создайте новый кластер PostgresSQL
docker-compose.yml
```yaml
version: '3.1'
services:
  server:
    image: postgres:14
    restart: always
    environment:
      POSTGRES_DB: task2
      POSTGRES_HOST_AUTH_METHOD: trust
    ports:
      - "5555:5432"
```

### зайдите в созданный кластер под пользователем postgres
```
dmitry.balabanov@MacBook-Pro-X5 task_3 % psql -U postgres -h localhost -p 5555
psql (14.5 (Homebrew), server 14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=# 
```

### создайте новую базу данных testdb
```
postgres=# create database testdb;
CREATE DATABASE
```

### зайдите в созданную базу данных под пользователем postgres
```
postgres=# \c testdb
psql (14.5 (Homebrew), server 14.4 (Debian 14.4-1.pgdg110+1))
You are now connected to database "testdb" as user "postgres".
testdb=# 
```

### создайте новую схему testnm
```
testdb=# create schema testnm;
CREATE SCHEMA
```

### создайте новую таблицу t1 с одной колонкой c1 типа integer
```
testdb=# create table t1 (c1 int); 
CREATE TABLE
```

### вставьте строку со значением c1=1
```
testdb=# insert into t1 (c1) values (1);
INSERT 0 1
```

### создайте новую роль readonly
```
testdb=# create role readonly login;
CREATE ROLE
```

### дайте новой роли право на подключение к базе данных testdb
```
testdb=# grant connect on database testdb to readonly;
GRANT
```

### дайте новой роли право на использование схемы testnm
```
testdb=# grant usage on schema testnm to readonly;
GRANT
```

### дайте новой роли право на select для всех таблиц схемы testnm
```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

### создайте пользователя testread с паролем test123
```
testdb=# create role testread login password 'test123';
CREATE ROLE
```

### дайте роль readonly пользователю testread
```
testdb=# grant readonly to testread;
GRANT ROLE
```

### зайдите под пользователем testread в базу данных testdb
```
dmitry.balabanov@MacBook-Pro-X5 task_3 % PGPASSWORD=test123 psql -U testread -h localhost -p 5555 testdb
psql (14.5 (Homebrew), server 14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

testdb=> 
```

### сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```

### получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
### напишите что именно произошло в тексте домашнего задания
### у вас есть идеи почему? ведь права то дали?
### посмотрите на список таблиц
### подсказка в шпаргалке под пунктом 20
### а почему так получилось с таблицей

Права на чтение из таблиц были выданы роли readonly для схемы testnm.
Таблица t1 принадлежит схеме public, на которую у роли readonly прав нет.

```
testdb=# select * from information_schema.role_table_grants where table_name = 't1';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy 
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | postgres | testdb        | public       | t1         | INSERT         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | SELECT         | YES          | YES
 postgres | postgres | testdb        | public       | t1         | UPDATE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | DELETE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRUNCATE       | YES          | NO
 postgres | postgres | testdb        | public       | t1         | REFERENCES     | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRIGGER        | YES          | NO
(7 rows)
```

### вернитесь в базу данных testdb под пользователем postgres
### удалите таблицу t1

```
testdb=# drop table t1;
DROP TABLE
```

### создайте ее заново но уже с явным указанием имени схемы testnm
### вставьте строку со значением c1=1

```
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1
```

### зайдите под пользователем testread в базу данных testdb
### делайте select * from testnm.t1;
### получилось?
```
testdb=> select * from testnm.t1 ;
ERROR:  permission denied for table t1
```

### есть идеи почему? если нет - смотрите шпаргалку
Отсутствуют права на чтение. 
Таблица testnm.t1 была создана после выполнения команды `grant select on all tables in schema testnm to readonly;`.

### как сделать так чтобы такое больше не повторялось?
```
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```

### теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```
dmitry.balabanov@MacBook-Pro-X5 task_3 % PGPASSWORD=test123 psql -U testread -h localhost -p 5555 testdb
psql (14.5 (Homebrew), server 14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```

### а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
В запросе на создание таблицы t2 явно не указана схема, т.е таблица создается в схеме public,
на которую у всех пользователей есть права.

### есть идеи как убрать эти права? если нет - смотрите шпаргалку
### если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
```
testdb=# revoke create on schema public from public;
REVOKE
testdb=# revoke all on all tables in schema public from public, readonly;
REVOKE
testdb=# grant select on all tables in schema public to readonly;
GRANT
```

### теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
INSERT 0 1
```
Вставка в t2 произведена, поскольку пользователь testread является ее владельцем  
```
testdb=# alter table t2 owner to postgres;
ALTER TABLE
```
```
testdb=> insert into t2 values (111);
ERROR:  permission denied for table t2
testdb=> select count(*) from t2;
 count 
-------
     9
(1 row)
```
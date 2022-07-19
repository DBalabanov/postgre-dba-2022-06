# Лекция 3: Установка PostgreSQL 

## Домашнее задание. Выполнил Балабанов Дмитрий.

### сделать в GCE инстанс с Ubuntu 20.04 или развернуть докер любым удобным способом
### поставить на нем Docker Engine

Docker предустановлен на локальной машине

```
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker --version && docker-compose --version
Docker version 20.10.11, build dea9396
docker-compose version 1.29.2, build 5becea4c
dmitry.balabanov@MacBook-Pro-X5 task_2 % 
```

### сделать каталог /var/lib/postgres

Создана директория *var_lib_postgresql* в каталоге с данным заданием

```
dmitry.balabanov@MacBook-Pro-X5 task_2 % mkdir var_lib_postgresql
dmitry.balabanov@MacBook-Pro-X5 task_2 % ls -la ./var_lib_postgresql 
total 0
drwxr-xr-x  2 dmitry.balabanov  staff   64 Jul 19 20:11 .
drwxr-xr-x  6 dmitry.balabanov  staff  192 Jul 19 20:11 ..
dmitry.balabanov@MacBook-Pro-X5 task_2 % 
```

### развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
### развернуть контейнер с клиентом postgres

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
    volumes:
      - ./var_lib_postgresql:/var/lib/postgresql/data

  client:
    build:
      context: .
    tty: true
    depends_on:
      - server
```

Dockerfile для клиента postgres:

```dockerfile
FROM ubuntu:20.04
RUN apt update && apt install --no-install-recommends -y postgresql-client
```

### подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
```
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker exec -it task_2_client_1 bash

root@dba86a77373d:/#
root@dba86a77373d:/# psql -U postgres -h task_2_server_1 -d task2
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

task2=# CREATE table task_2 (data text);
CREATE TABLE
task2=# INSERT INTO task_2 (data) VALUES ('data1'), ('data2');
INSERT 0 2
task2=# SELECT * FROM task_2;
 data  
-------
 data1
 data2
(2 rows)

task2=# 
```

### подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/места установки докера
```
dmitry.balabanov@MacBook-Pro-X5 task_2 % psql -U postgres -h localhost -p 5555 task2
psql (14.1, server 14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

task2=#  SELECT * FROM task_2;
 data  
-------
 data1
 data2
(2 rows)

task2=# 
```

### удалить контейнер с сервером
```
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker stop task_2_server_1 
task_2_server_1
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker rm task_2_server_1 
task_2_server_1
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker ps -a           
CONTAINER ID   IMAGE           COMMAND       CREATED          STATUS          PORTS     NAMES
3a264802be25   task_2_client   "/bin/bash"   12 minutes ago   Up 12 minutes             task_2_client_1
dmitry.balabanov@MacBook-Pro-X5 task_2 % 
```

### создать его заново
```
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker-compose up server
Creating task_2_server_1 ... done
Attaching to task_2_server_1
...
server_1  | 2022-07-19 18:12:15.281 UTC [1] LOG:  database system is ready to accept connections
```

### подключится снова из контейнера с клиентом к контейнеру с сервером
### проверить, что данные остались на месте
```
dmitry.balabanov@MacBook-Pro-X5 task_2 % docker exec -it task_2_client_1 bash
root@3a264802be25:/# psql -U postgres -h task_2_server_1 -d task2
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

task2=# SELECT * FROM task_2;
 data  
-------
 data1
 data2
(2 rows)

task2=# 
```
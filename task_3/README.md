# Физический уровень PostgreSQL

---
**_Примечание_**: Домашнее задание выполнялось без использования сервисов GCE.

В качестве виртуальных машин использовались docker-контейнеры с примонтированной папкой в качестве внешнего диска. 

---

### создайте виртуальную машину c Ubuntu
### поставьте на нее PostgreSQL 14
Dockerfile
```dockerfile
FROM ubuntu:22.04
ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt install --no-install-recommends -y postgresql mc
```

docker-compose.yaml
```yaml
version: '3.1'
services:
  postgres:
    build:
      context: .
    container_name: postgres
    tty: true
    volumes:
      - ./mnt_data:/mnt/data
```

Результат:
```
dmitry.balabanov@MacBook-Pro-X5 task_3 % docker-compose up
Creating network "task_3_default" with the default driver
Creating postgres ... done
Attaching to postgres
```

### проверьте что кластер запущен
```
dmitry.balabanov@MacBook-Pro-X5 task_3 % docker exec -it postgres bash                                                                                                             
root@ded9f3ec55da:/# su postgres
postgres@ded9f3ec55da:/$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
postgres@ded9f3ec55da:/$ service postgresql start
 * Starting PostgreSQL 14 database server                                                                                                                                                                                        [ OK ] 
postgres@ded9f3ec55da:/$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
postgres@ded9f3ec55da:/$
```

### зайдите из-под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```
postgres@ded9f3ec55da:/$ psql 
psql (14.5 (Ubuntu 14.5-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# create table test (c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# 
 ```

### остановите postgres
```
postgres@ded9f3ec55da:/$ service postgresql stop
 * Stopping PostgreSQL 14 database server                                                                                                                                                                                        [ OK ] 
postgres@ded9f3ec55da:/$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

### проинициализируйте диск согласно инструкции и подмонтировать файловую систему
Файловая система смонтирована в /mnt/data при инициализации контейнера

### сделайте пользователя postgres владельцем /mnt/data
```
root@ded9f3ec55da:/# chown postgres:postgres /mnt/data
```

### перенесите содержимое /var/lib/postgres/14 в /mnt/data
```
root@ded9f3ec55da:/# mv /var/lib/postgresql/14/main /mnt/data
root@ded9f3ec55da:/# ls -l /mnt/data/main/
total 12
-rw-------  1 postgres postgres    3 Oct 25 20:29 PG_VERSION
drwx------  5 postgres postgres  160 Oct 25 20:29 base
drwx------ 60 postgres postgres 1920 Oct 25 20:33 global
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_commit_ts
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_dynshmem
drwx------  5 postgres postgres  160 Oct 25 20:34 pg_logical
drwx------  4 postgres postgres  128 Oct 25 20:29 pg_multixact
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_notify
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_replslot
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_serial
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_snapshots
drwx------  6 postgres postgres  192 Oct 25 20:34 pg_stat
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_stat_tmp
drwx------  3 postgres postgres   96 Oct 25 20:29 pg_subtrans
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_tblspc
drwx------  2 postgres postgres   64 Oct 25 20:29 pg_twophase
drwx------  4 postgres postgres  128 Oct 25 20:29 pg_wal
drwx------  3 postgres postgres   96 Oct 25 20:29 pg_xact
-rw-------  1 postgres postgres   88 Oct 25 20:29 postgresql.auto.conf
-rw-------  1 postgres postgres  130 Oct 25 20:32 postmaster.opts
root@ded9f3ec55da:/# ls -l /var/lib/postgresql/14/main
ls: cannot access '/var/lib/postgresql/14/main': No such file or directory
```

### попытайтесь запустить кластер
```
root@ded9f3ec55da:/# su postgres
postgres@ded9f3ec55da:/$ pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```

### напишите получилось или нет и почему
Ошибка! Нет доступа к данным на уровне файловой системы.

### задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его
```
postgres@ded9f3ec55da:/$ cat /etc/postgresql/14/main/postgresql.conf | grep data_dir
data_directory = '/mnt/data/main'               # use data in another directory
```

### попытайтесь запустить кластер
```
postgres@ded9f3ec55da:/$ pg_ctlcluster 14 main start
postgres@ded9f3ec55da:/$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory Log file
14  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-14-main.log
```

### напишите получилось или нет и почему
Директория с данными найдена. Сервер запустился.

### зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
postgres@ded9f3ec55da:/$ psql
psql (14.5 (Ubuntu 14.5-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# select * from test;
 c1 
----
 1
(1 row)
```


## Задание со звездой
Не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

docker-compose.star.yml
```yaml
version: '3.1'
services:
  postgres2:
    build:
      context: .
    container_name: postgres2
    tty: true
    volumes:
      - ./mnt_data:/mnt/data
```

```
dmitry.balabanov@MacBook-Pro-X5 task_3 % docker-compose -f docker-compose.star.yml up 

dmitry.balabanov@MacBook-Pro-X5 task_3 % docker ps
CONTAINER ID   IMAGE              COMMAND   CREATED          STATUS          PORTS     NAMES
7be6a1b62662   task_3_postgres2   "bash"    30 seconds ago   Up 29 seconds             postgres2
ded9f3ec55da   task_3_postgres    "bash"    8 minutes ago    Up 4 minutes              postgres

dmitry.balabanov@MacBook-Pro-X5 task_3 % docker exec -it postgres2 bash 
root@7be6a1b62662:/# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@7be6a1b62662:/# cat /etc/postgresql/14/main/postgresql.conf | grep data_dir
data_directory = '/var/lib/postgresql/14/main'          # use data in another directory

root@7be6a1b62662:/# mcedit /etc/postgresql/14/main/postgresql.conf                

root@7be6a1b62662:/# cat /etc/postgresql/14/main/postgresql.conf | grep data_dir
data_directory = '/mnt/data/main'               # use data in another directory

root@7be6a1b62662:/# su postgres
postgres@7be6a1b62662:/$ pg_ctlcluster 14 main start
postgres@7be6a1b62662:/$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory Log file
14  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-14-main.log

postgres@7be6a1b62662:/$ psql
postgres=# select * from test;
 c1 
----
 1
(1 row)
```
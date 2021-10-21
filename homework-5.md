Домашнее задание
Работа с базами данных, пользователями и правами

Цель:
создание новой базы данных, схемы и таблицы
создание роли для чтения данных из созданной схемы созданной базы данных
создание роли для чтения и записи из созданной схемы созданной базы данных
1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)

    slava@homework-5:~$ pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log

2 зайдите в созданный кластер под пользователем postgres
3 создайте новую базу данных testdb
4 зайдите в созданную базу данных под пользователем postgres
5 создайте новую схему testnm
6 создайте новую таблицу t1 с одной колонкой c1 типа integer
7 вставьте строку со значением c1=1
8 создайте новую роль readonly

    postgres=# create database testdb;
    CREATE DATABASE
    postgres=# \c testdb
    You are now connected to database "testdb" as user "postgres".
    testdb=# create schema testnm;
    CREATE SCHEMA
    testdb=# create table t1(c1 int);
    CREATE TABLE
    testdb=# create role readonly;
    CREATE ROLE

9 дайте новой роли право на подключение к базе данных testdb

    testdb=# alter user readonly login;
    ALTER ROLE
   
    Тут не совсем понятно как давать права на подключение только к testdb. По ALTER user LOGIN даём права на все базы. 
    Ещё можем задать права правкой pg_hba.conf. Может это имеется ввиду.
    
    Нашёл - grant CONNECT on DATABASE testdb to readonly;

10 дайте новой роли право на использование схемы testnm
11 дайте новой роли право на select для всех таблиц схемы testnm

    testdb=# grant USAGE on SCHEMA testnm to readonly;
    GRANT
    testdb=# grant SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
    GRANT

12 создайте пользователя testread с паролем test123
13 дайте роль readonly пользователю testread
14 зайдите под пользователем testread в базу данных testdb

    postgres=# create user testread PASSWORD 'test123';
    CREATE ROLE
    postgres=# grant readonly TO testread;
    GRANT ROLE
    postgres=# \du
                                    List of roles
    Role name |                         Attributes                         | Member of
    -----------+------------------------------------------------------------+------------
    postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
    readonly  |                                                            | {}
    testread  |                                                            | {readonly}
    \q    
    slava@homework-5:~$ psql -U testread -d testdb -h localhost
    Password for user testread:
    psql (13.4 (Debian 13.4-4.pgdg100+1))
    SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
    Type "help" for help.

    testdb=>

15 сделайте select * from t1;
16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

    testdb=> select * from t1;
    ERROR:  permission denied for table t1

17 напишите что именно произошло в тексте домашнего задания
18 у вас есть идеи почему? ведь права то дали?
19 посмотрите на список таблиц

    Права мы дали на таблицы схемы testnm, а таблица t1 в схеме public
    testdb=> \d
            List of relations
     Schema | Name | Type  |  Owner
    --------+------+-------+----------
     public | t1   | table | postgres
    (1 row)

20 подсказка в шпаргалке под пунктом 20
21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
22 вернитесь в базу данных testdb под пользователем postgres
23 удалите таблицу t1
24 создайте ее заново но уже с явным указанием имени схемы testnm
25 вставьте строку со значением c1=1
26 зайдите под пользователем testread в базу данных testdb
27 сделайте select * from testnm.t1;
28 получилось?
29 есть идеи почему? если нет - смотрите шпаргалку

    Не получилось потому что после добавбления прав на все таблицы схемы мы пересоздали таблицу.

    slava@homework-5:~$ sudo -u postgres psql
    psql (13.4 (Debian 13.4-4.pgdg100+1))
    Type "help" for help.

    postgres=# \c testdb postgres
    You are now connected to database "testdb" as user "postgres".
    testdb=# drop table t1;
    DROP TABLE
    testdb=# create table testnm.t1(c1 int);
    CREATE TABLE
    testdb=# insert into testnm.t1 values (1);
    INSERT 0 1
    testdb=# \q
    slava@homework-5:~$ psql -U testread -d testdb -h localhost
    Password for user testread:
    psql (13.4 (Debian 13.4-4.pgdg100+1))
    SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
    Type "help" for help.

    testdb=> select * from testnm.t1;
    ERROR:  permission denied for table t1
    testdb=>

30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

    Нагуглил что нужно установить права по умолчанию )

    postgres=# \c testdb
    You are now connected to database "testdb" as user "postgres".
    testdb=# alter default privileges in schema testnm GRANT SELECT ON TABLES TO readonly;
    ALTER DEFAULT PRIVILEGES
    testdb=# drop table testnm.t1 ;
    DROP TABLE
    testdb=# create table testnm.t1(c1 int);
    CREATE TABLE
    testdb=# insert into testnm.t1 values (1);
    INSERT 0 1
    testdb=# \q
    slava@homework-5:~$ psql -U testread -d testdb -h localhost
    Password for user testread:
    psql (13.4 (Debian 13.4-4.pgdg100+1))
    SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
    Type "help" for help.

    testdb=> select * from testnm.t1;
     c1
    ----
      1
    (1 row)

31 сделайте select * from testnm.t1;
32 получилось?
33 есть идеи почему? если нет - смотрите шпаргалку
31 сделайте select * from testnm.t1;
32 получилось?
33 ура!
34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

    testdb=> create table t2(c1 integer); insert into t2 values (2);
    CREATE TABLE
    INSERT 0 1

35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

    Таблицу t2 создали в схеме public, а там по умолчанию права на создание есть. Все юзеры неявно сотоят в роли public (где-то читал)

36 есть идеи как убрать эти права? если нет - смотрите шпаргалку

    Надо отозвать права из схемы public:
    REVOKE CREATE ON SCHEMA public FROM public;
    

37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды

    Шпаргалку счас посмотрел, вроде всё так и сам выше написал.

38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39 расскажите что получилось и почему 

    testdb=> create table t3(c1 integer); insert into t2 values (2);
    ERROR:  permission denied for schema phomework-5ublic
    
    Теперь нет прав на схему public через роль public.
    
 # Project OTUS-Postgres
 # Instance homework-5
 

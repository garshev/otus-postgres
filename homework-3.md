Google project - OTUS-Postgres

Домашнее задание
Установка и настройка PostgreSQL

Цель:
создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
переносить содержимое базы данных PostgreSQL на дополнительный диск
переносить содержимое БД PostgreSQL между виртуальными машинами
создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a
поставьте на нее PostgreSQL через sudo apt
проверьте что кластер запущен через sudo -u postgres pg_lsclusters

    slava@homework-3:~$ sudo -u postgres pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
   
зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым postgres=# create table test(c1 text); postgres=# insert into test values('1'); \q
остановите postgres например через sudo -u postgres pg_ctlcluster 13 main stop
создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
перенесите содержимое /var/lib/postgres/13 в /mnt/data - mv /var/lib/postgresql/13 /mnt/data
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
напишите получилось или нет и почему

    Не получилось, сервис postgres не нашёл установленный в конфиге /var/lib/postgresql/12/main

задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его
напишите что и почему поменяли

    Установили правильный путь для: data_directory = '/mnt/data/12/main'

попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
напишите получилось или нет и почему

    Кластер запустился

зайдите через через psql и проверьте содержимое ранее созданной таблицы

    postgres=# select * from test;
    c1
    ----
    1
    (1 row)

задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

    
    Чтобы не отключать дополнительный диск от первой машины я его склонировал и подключил ко второй машине homework-3-add. Потом как и в первой машине примонтировал к /mnt/data, остановил postgres, удалил /var/lib/postgresql, поправил postgresql.conf для указания нового пути: data_directory = '/mnt/data/12/main'. Сервис запустился нормально, данные на месте:
    postgres=# select * from test;
    c1
    ----
    1
    (1 row)


    
ДЗ оформите в markdown на github с описанием что делали и с какими проблемами столкнулись. Также приложите имя Гугл проекта, где пользователь ifti@yandex.ru добавлен в роли project editor.

    Google project - OTUS-Postgres

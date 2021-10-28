Настройте выполнение контрольной точки раз в 30 секунд.

    sed -i 's/#checkpoint_timeout = 5min/checkpoint_timeout = 30s/' /etc/postgresql/11/main/postgresql.conf
    grep 'checkpoint' /etc/postgresql/11/main/postgresql.conf
    egrep 'checkpoint_timeout|log_checkpoints' /etc/postgresql/11/main/postgresql.conf
    checkpoint_timeout = 30s                # range 30s-1d
    log_checkpoints = on
    postgres@homework-7:~/11/main$ pg_ctlcluster 11 main reload
    
10 минут c помощью утилиты pgbench подавайте нагрузку.

    postgres@homework-7:~/11/main$ pgbench -T 600
    tps = 311.746542 (including connections establishing)

Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

    postgres@homework-7:~$ grep 'distance=' /var/log/postgresql/postgresql-11-main.log | awk '{ buffers += $9 } END { print buffers }'
    38628
    
    Было записано 38628 буферов. Общиё объём 38628 * 8кБ ~ 300 MB. Вс реднем по 16 МБ за контрольную точку.

Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

    Все точки ровно через 30 сек кроме последней. Она через 60 сек.
    Пока нет идей почему так.
    
Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

    В асинхронном режиме tps вырос с 311 до 836. В асинхронном режиме не выполняется системный вызов sync при каждом коммите - отсюда скорость.

Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. 
Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

    pg_createcluster 11 sumon -- --data-checksums
    
    postgres@homework-7:~$ psql -p 5433
    psql (11.13 (Debian 11.13-0+deb10u1))
    Type "help" for help.

    postgres=# select pg_relation_filepath('t1');
     pg_relation_filepath
    ----------------------
     base/13100/16384
    (1 row)
    
    postgres@homework-7:~$ dd if=/dev/zero of=/var/lib/postgresql/11/sumon/base/13100/16384 oflag=dsync conv=notrunc bs=1 count=8
    
    postgres=# select * from t1 ;
    WARNING:  page verification failed, calculated checksum 5279 but expected 22515
    ERROR:  invalid page in block 0 of relation base/13100/16384
    
    Обнаружена ошибка при проверке контрольных сум.
    
    postgres=# alter system set ignore_checksum_failure TO on;
    ALTER SYSTEM
    
    postgres=# select pg_reload_conf();
    -[ RECORD 1 ]--+--
    pg_reload_conf | t

    postgres=# select * from t1 ;
    WARNING:  page verification failed, calculated checksum 5279 but expected 22515
    -[ RECORD 1 ]
    i | 1
    -[ RECORD 2 ]
    i | 2
